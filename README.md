# Drift: Cross-Platform Input Continuity

Project **Drift** is an ambitious, open-source development project designed to bridge the ecosystem gap between desktop operating systems (Apple macOS / Microsoft Windows) and Google Android. Its primary objective is to create a lightweight, robust, hardware-independent "Universal Control" pipeline that allows a single mouse and keyboard to cross operating system boundaries seamlessly to control an Android target.

## Project Codebases (Git Submodules)
* 🍏 **macOS Transmitter (Swift / SwiftUI):** [AirDrift](https://github.com/Falak-Parmar/AirDrift)
* 🤖 **Android Receiver (Kotlin / Jetpack Compose):** [DroidDrift](https://github.com/Falak-Parmar/DroidDrift)

---

## 1. The Core Vision
Unlike official ecosystem tools that restrict layout topology or lock software to a single brand ecosystem, Drift provides a unified, cross-platform workspace. By establishing **DroidDrift** as the central target hub, a user can transition control dynamically from their primary computer, regardless of whether it is a Mac or a Windows PC.

### Multi-Screen Chaining Topology Examples:
* **The Apple/Android Chain:** The cursor moves across the MacBook display, tunnels through an iPad via native Apple Universal Control, exits the edge of the iPad, and instantly wakes up the native Android pointer on a phone or tablet.
```
[ MacBook (AirDrift) ] -------> [ iPad (Native) ] -------> [ Android Device (DroidDrift) ]
```

* **The PC Desktop Chain:** The cursor hits the boundary of a multi-monitor Windows workstation setup and seamlessly warps directly onto a side-car Android tablet.
```
[ Windows PC (WinDrift) ] ------------------------------> [ Android Device (DroidDrift) ]
```

---

## 2. Technical Architecture & System APIs
To bypass fragile, closed-source system frameworks, Drift environments run entirely in user space utilizing rock-solid network sockets, accessibility permissions, and official operating system input hooks.

```
+-------------------------------------------------------------+
|                     macOS (AirDrift)                        |
|  +-------------------------------------------------------+  |
|  |                CGEventTap Interception                |  |
|  +---------------------------+---------------------------+  |
|                              | (Deltas / Coordinates)        |
|                              v                              |
|  +-------------------------------------------------------+  |
|  |                  WebSocket Client                     |  |
|  +---------------------------+---------------------------+  |
+------------------------------|------------------------------+
                               | (JSON Stream via Wi-Fi/USB)
                               v [Port 8080]
+-------------------------------------------------------------+
|                 Android Device (DroidDrift)                 |
|                                                             |
|  +-------------------------------------------------------+  |
|  |     InputAccessibilityService (WebSocket Server)     |  |
|  |  - Port 8080: Ingests macOS mouse/keyboard events     |  |
|  |  - Updates floating UI cursor position overlay        |  |
|  |  - Tracks boundary exits & temporal cooldowns         |  |
|  +---------------------------+---------------------------+  |
|                              |                              |
|                              +-----------------------+      |
|                              | (Fallback Mode)       | (TCP) | [Port 9000]
|                              v                       v      |
|                +-------------+------------+    +-----+-----+|
|                | Accessibility Click/Drag |    | AdbMain   ||
|                |   (dispatchGesture API)  |    | (Daemon)  ||
|                +--------------------------+    +-----+-----+|
|                                                      |      |
|                                                      v      |
|                                                +-----+-----+|
|                                                | IInputMgr ||
|                                                | Binder    ||
|                                                | Injection ||
|                                                +-----------+|
+-------------------------------------------------------------+
```

### The macOS Transmitter: `AirDrift` (Swift / SwiftUI)
* **Input Interception (`CGEventTap`):** Intercepts low-level physical mouse movements (`mouseMoved`) and keyboard states (`keyDown`/`keyUp`) before they hit the window server.
* **Edge Locking:** Swallows the local system cursor once it hits the virtual boundary, freezing it in place while translating raw relative movement vectors into structured JSON tracking payloads.
* **Lock Cooldown:** Incorporates a 500ms lock cooldown to prevent pending queue events from immediately re-locking the cursor when returning control to the Mac, resolving recursive looping.

### The Android Hub Receiver: `DroidDrift` (Kotlin / Jetpack Compose)
Rather than a single server causing port conflicts, DroidDrift uses a dual-mode, host-client architecture:
1. **`InputAccessibilityService` (App-Space WebSocket Server - Port 8080):** 
   - Receives events from the macOS transmitter.
   - Manages absolute screen coordinates and renders a beautiful iPadOS-style circular cursor overlay.
   - Triggers the edge-exit transition to restore Mac cursor focus with a 500ms exit cooldown.
2. **`AdbMain` (Privileged ADB-Shell Daemon - Port 9000):**
   - Bypasses standard Android restriction on `INJECT_EVENTS` by running in the ADB shell context (`UID 2000`) using Android's `app_process` command.
   - Connects to the local binder `"input"` service via `ServiceManager.getService("input")` to retrieve `IInputManager` reflection-free.
   - Receives action payloads from `InputAccessibilityService` over a local TCP loopback (port 9000) and injects touch/pointer events at the driver level.
3. **Dual-Mode Graceful Fallback:** 
   - If the ADB daemon is not running (e.g., disconnected USB/wireless debug), the App-Space service automatically falls back to simulating clicks and scrolls globally using the accessibility service's `dispatchGesture()` API.

---

## 3. Visual Pointer & User Interface
* **iPadOS-Style Floating Cursor:** Uses the `SYSTEM_ALERT_WINDOW` permission to draw a beautiful, modern, translucent blue circular pointer with a white border and drop shadow. This cursor is overlayed on top of all Android apps and updates dynamically, solving the issue of One UI hiding the native pointer when no physical USB mouse is plugged in.
* **Glassmorphic Settings Setup:** The Jetpack Compose UI has been overhauled to present a premium glassmorphic checklist screen. It dynamically detects permission states, displays real-time connection info (local IP, socket state), and guides the user through enabling overlay and accessibility services.

---

## 4. Branding & Directory Architecture
* **macOS App:** `AirDrift` (Wireless, lightweight background utility for Mac)
* **Android App:** `DroidDrift` (The Android accessibility service & privileged injector daemon)

---

## 5. Startup & Usage Checklist

1. **Build and Install DroidDrift APK:**
   ```bash
   adb push DroidDrift/app/build/outputs/apk/debug/app-debug.apk /data/local/tmp/
   ```
2. **Grant Permissions:**
   - Open DroidDrift and toggle the glassmorphic checklist settings to grant **Display Over Other Apps** (Overlay) and enable the **DroidDrift Accessibility Service**.
3. **Spawn Privileged ADB Daemon (Optional but Recommended for full injection):**
   ```bash
   adb shell "export CLASSPATH=/data/local/tmp/app-debug.apk; exec app_process /data/local/tmp com.drift.droiddrift.AdbMain 1080 2320"
   ```
4. **Port Forward & Run macOS Transmitter:**
   ```bash
   adb forward tcp:8080 tcp:8080
   cd AirDrift
   swift run AirDrift localhost
   ```
