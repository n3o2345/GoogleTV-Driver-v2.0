# AndroidTV Control4 Driver v2.0

A Control4 DriverWorks driver for controlling Android TV / Google TV devices using the **Android TV Remote Service v2** protocol over TCP — no ADB or Developer Options required for standard operation.

> **Credit:** Driver created by **Greg Moerler**. This repository is a distribution of his work.

---

## Overview

This driver integrates Android TV and Google TV devices into a Control4 OS 3.x smart home system. It communicates over a native TLS-encrypted TCP connection (port 6466 for commands, port 6467 for pairing), enabling full remote control, power management, app launching, and real-time feedback — all without requiring ADB debugging or Developer Options on the TV.

The included **AndroidTV_Tool** Windows utility handles two tasks:
- **Extract Public Key** — retrieves the RSA certificate from the device (required for pairing, no ADB needed)
- **Extract App List** — scans the device for all installed app package names (requires ADB to be enabled on the device; ADB can be turned back off once the list is captured)

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
    ├── TV_Toolkit.exe             # GUI tool for RSA key extraction and app list
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
- **Windows PC** (for one-time setup with `AndroidTV_Tool`)

---

## Installation

1. Copy `driver/Android_v2.0.c4z` to your Control4 driver directory or install via Composer.
2. Add the driver to your project and set the **network binding** to the Android TV device's IP address.
3. Proceed to [Pairing Setup](#pairing-setup).

---

## Pairing Setup

Pairing is a one-time process that exchanges RSA certificates with the device.

### Step 1 — Extract the Device Public Key

Launch **TV_Toolkit.exe** on a Windows PC on the same network as the Android TV device:

1. Enter the **Device IP Address**
2. Click the **Extract Public Key** tab
3. Click **Get Public Key**
4. The tool queries the device over TCP 6467 and writes the certificate info to `cert_info.txt`
5. Copy the **Modulus** and **Exponent** (typically `65537`) from the output

Alternatively, using OpenSSL manually from a command prompt:

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

## Extracting the App List

The **Extract App List** feature in `TV_Toolkit.exe` scans your Android TV device and generates a list of every installed app's package name — useful for populating the **Launch App URL** slots in the driver.

> **Requires ADB:** Developer Options and ADB Debugging must be enabled on the TV for this step. ADB can be disabled again once the list is captured.

### How to Enable ADB on the TV

1. Go to **Settings → Device Preferences → About**
2. Click **Build** repeatedly until Developer Options are enabled
3. Go to **Settings → Device Preferences → Developer Options**
4. Enable **USB Debugging** (ADB)
5. When prompted on the TV screen, approve the connection from your PC

### Extracting the App List

1. Launch **TV_Toolkit.exe**
2. Enter the **Device IP Address**
3. Click the **Extract App List** tab
4. Click **Scan — Generate apps.txt**
5. The tool connects via ADB, queries all installed packages, and saves the results to `apps.txt`

### Reading the Output

`apps.txt` lists one package name per line — one entry per installed app:

```
com.plexapp.android
com.peacocktv.peacockandroid
com.hulu.livingroomplus
tv.pluto.android
com.disney.disneyplus
```

### Entering App Package Names in Composer

In the driver's Properties panel, scroll to the **Launch App URLs** section. Copy package names from `apps.txt` and enter one per slot (up to 20):

```
Launch App URL 1  →  com.plexapp.android
Launch App URL 2  →  com.peacocktv.peacockandroid
Launch App URL 3  →  com.google.android.youtube.tv
```

### Creating Mini-Drivers for Each App

Once your package names are configured in the driver, use **[RJ Boucher's Control4 Mini-Driver Creator](https://mdc.rjboucher.com/)** to generate a Control4 mini-driver for each app. Mini-drivers give each app its own icon, name, and launch binding in the Control4 UI — making them selectable from the Navigator just like any other source.

1. Go to **https://mdc.rjboucher.com/**
2. Enter the app name and package name (e.g. `Plex` / `com.plexapp.android`)
3. Download the generated `.c4z` mini-driver
4. Install it in Composer and connect it to this driver's app launch binding

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
| Disney+ | `com.disney.disneyplus` |
| Hulu | `com.hulu.livingroomplus` |
| Prime Video | `com.amazon.amazonvideo.livingroom` |
| Peacock | `com.peacocktv.peacockandroid` |
| Plex | `com.plexapp.android` |
| Philo | `com.philo.philo.google` |
| Pluto TV | `tv.pluto.android` |
| Tubi | `com.tubitv` |
| Roku Channel | `com.roku.web.trc` |
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
| ADB connection refused | Enable Developer Options → ADB Debugging on the TV, then approve the prompt on screen |
| App list scan fails | Confirm ADB is enabled and the TV approved the connection from your PC |
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

ADB is only needed for the optional app list scan. All other functionality (pairing, control, app launching, current-app feedback) works without it.

---

## Credits

Driver authored by **Greg Moerler**.

Distributed with permission. All driver logic, certificate handling, and protocol implementation are his original work.
