This Repository Includes The Documentations As Well As The Source Code For The LED - IOT - Pichler Project





# Big Picture

It’s a **Raspberry Pi–based RGB solar lamp** stack (CPython + Adafruit Blinka). It controls:

* **RGB LEDs** (WS2812/NeoPixel patterns)
* **Two stepper/driver axes** (tilt + rotation) for lamp head positioning
* **Battery fuel-gauge** (MAX17261 over I²C), plus “solar charging / AC cable present” logic
* **Compass/IMU & GPS** (for orientation and location)
* **Sun tracking** (astronomical calc + compass heading)
* **Buttons & capacitive touch** (AT42QT I²C)
* **Wi-Fi & BLE** status/management
* **Local web server** via Microdot (REST/WebSocket control, status, presets)
* **Update scheduler** (pulls updates from a Git repo)

---

# Entry Points & Orchestration (run these)

* **`firmware-quellcode/main.py`**
  Boots the system, normalizes JSON config files, instantiates the **MotorInterpreter** and the LED manager, wires buttons/touch, and ties everything into the current lamp model (dimming, modes, schedules). It’s effectively the **runtime supervisor**.

* **`firmware-quellcode/webserver_microdot.py`**
  Microdot web app: HTTP/WebSocket endpoints to read status, pick LED programs, toggle glow/dimming, start/stop motors, etc. Think **local control panel API**.

* **`firmware-quellcode/update_manager.py`**
  Background “check for update and apply” component (uses `requests`, `schedule`, and shell git calls). Hard-coded base path: `/home/Solarlampe/Solarlamp_on_raspi`.

---

# Motion / Mechatronics

* **`control_motor_interpreter.py`** → **MotorInterpreter**
  The heart of motion logic. Coordinates both axes, owns locks/threads, exposes high-level actions (go to angle, continuous rotation, tracking start/stop), and reads/writes `current_motor_status.json`.

* **`tilt_motor_control.py`** and **`rotation_motor_control.py`**
  Low-level GPIO step/dir/enable control using `board` + `digitalio`. Implements steps-per-deg, speed, enable/disable, and position tracking. Uses `shared_ressources.SharedResources` stop flags.

* **`control_sun_tracking.py`**
  Uses **location + time + compass** to compute the sun azimuth/elevation and feeds MotorInterpreter to track the sun.

* **`control_compass.py` / `Kompass_Accelerometer.py`**
  I²C magnetometer/accelerometer readout (via `busio` + Adafruit I²C device). Provides heading and raw axes.

---

# LEDs (effects/programs)

* **`led_programms/`**
  Each file is one visual effect using `neopixel` + `board`:

  * `led_programm_rainbow.py`, `..._xmas.py`, `..._relax.py`, `..._galaxy.py`, `..._rainy_days.py`, `..._pulsar.py`, `..._earth.py`, `..._sleeping_motion.py`, `..._crazy_alien.py`, `..._indigo_cure.py`
  * `led_programm_benutzerdefiniert.py` = **user-defined** colors/steps.
  * `led_programm_bluetooth_active.py` = BLE status effect.

* **`led_programmliste.py`** → **LEDProgramManager**
  Keeps a registry of programs, starts/stops the active effect in a separate process, and **auto-dimming** integration (based on battery SOC/cable). Also starts a motor demo when requested.

* **`lamp_demo_threads.py`**
  Demo movements for motors (e.g., showroom mode).

* **`led_disable_on_day.py` / `led_laufzeiten_scheduler.py`**
  Scheduling & day-disable helpers—used to gate effects on time windows.

---

# Power / Battery / Charging

* **`control_ladechip.py`** → **MAX17261**
  I²C fuel gauge driver: init/config, SOC/voltage/current/temperature reads, and background monitoring (detect fast SOC jumps). Used by LED manager for dimming and by the web UI for status.

* **`check_ist_am_laden.py`**
  Determines whether the lamp is charging / cable is present (affects dimming and effect restarts).

---

# Inputs: Buttons & Touch

* **`SW_Button.py`**
  Physical button handling (debounce, long/short press patterns) with hooks into program changes or power states.

* **`control_top_button.py` / `control_bottom_button.py`**
  Integrate specific pins or AT42QT capacitive touch (via `i2c_devices_new/at42qt`) to map user gestures to actions (mode change, glow toggle, etc.).

* **`i2c_devices_new/at42qt/…`**
  Vendored AT42QT1070 I²C touch library and examples.

---

# Position/Location

* **`control_gps.py`**
  GPS reading logic (serial), feeds position/time to sun tracking.

* **`sonnenposition.py` / `sonnenstand.py`**
  Astronomical helpers (sun position math).

---

# Connectivity & OS Integrations

* **`WIFI_Manager.py`**
  Wi-Fi utilities: connectivity checks, IP detection (parsing `iwconfig` / `ip addr`), and network status.

* **`Bluetooth_LE.py`**
  Manages BLE adapter state via `rfkill` + `bluezero`. Triggers the LED “BLE active” program while advertising is on.

* **`microdot_*.py`**
  Vendored Microdot framework (ASGI/WSGI, websockets, sessions, templates). This lets you run the web UI/API without external install.

* **`timesync.py`**
  Enables NTP via `timedatectl` and logs a “time synced” stamp.

* **`change_host.py` / `set_hostname.py` / `new_hostname.py`**
  Utilities to set the Raspberry Pi hostname (the main code calls `checkIfDefaultHostNameAndChange`).

---

# Configuration & State (JSON)

* **`current_lampmodel.json`**
  Main **runtime state/config**: program (`modus`), glow toggles, **dimming values**, selected custom colors, schedule slots, Wi-Fi IP, tracking flags, etc.

* **`current_lampmodel_shared.json`**
  Shared subset of lamp model (for quick UI exchange).

* **`check_lampmodel.json`**
  “Schema” / defaults used to backfill missing keys on boot.

* **`current_motor_status.json`** & **`check_motor_status.json`**
  Actual vs. default “schema” for motor state (desired/current angles, speed, auto flags, etc.).

* **`motor_control_test.json`, `motor_control_preview.json`, `Update_Info.json`**
  Test/preview configs and OTA meta.

---

# Legacy / Reference

* **`old/`** and subfolders (`old/ESP32`, `beta`, `primitives`, etc.)
  Large archive of earlier experiments (including ESP32 variants). Useful for reference but **not part of the active runtime**.

---

# How it fits together (runtime flow)

1. **Boot:** `main.py` ensures JSON defaults exist, sets hostname if default, creates **MotorInterpreter** + **LEDProgramManager**, wires **buttons/touch**, and starts auxiliary threads (battery monitor, scheduler).
2. **Web UI/API:** `webserver_microdot.py` serves controls for LED mode, dimming, glow toggles, motor jog/angles, and returns live status via REST/WebSocket.
3. **Sensors & Power:** `control_ladechip.py` reads SOC/current/voltage; `check_ist_am_laden.py` checks for cable/charging. LED manager auto-dims or restarts effects when thresholds change.
4. **Tracking:** When enabled, `control_sun_tracking.py` uses **GPS + compass** to compute the sun heading, then MotorInterpreter orients tilt/rotation accordingly.
5. **Connectivity:** `WIFI_Manager.py` tracks Wi-Fi status; `Bluetooth_LE.py` can enable BLE and show a special LED pattern while advertising.

---

# How to run (typical dev setup on Raspberry Pi)

1. **Enable Blinka + GPIO**: Raspberry Pi OS with `python3`, `pip`, `I²C` and `SPI` enabled (`raspi-config`).
2. **Install deps (approx)**:

   ```bash
   sudo apt install python3-pip python3-dev libglib2.0-dev
   pip3 install adafruit-blinka adafruit-circuitpython-neopixel adafruit-circuitpython-mpr121 bluezero schedule pydbus
   ```

   (You already vendor **Microdot** modules, so no need to pip-install it.)
3. **Wire hardware**: NeoPixel DIN to a supported GPIO (per program), step/dir/enable pins per `tilt_motor_control.py` & `rotation_motor_control.py`, I²C for MAX17261 + AT42QT, magnetometer, and GPS on UART.
4. **Run**:

   ```bash
   cd firmware-quellcode
   python3 main.py
   ```

   Then point a browser to the device’s IP (Microdot webserver) to control modes/motors.

---

# Risks / Quick Wins

* **Hard-coded paths & privileges**
  `update_manager.py` and `timesync.py` use `/home/Solarlampe/...` and `sudo timedatectl`. Consider wrapping privileged calls behind a service script and making base paths **configurable**.
* **Requirements file**
  `requirements.txt` is not pip-compatible (includes stdlib names + commas). Replace with an actual pip list (`bluezero`, `schedule`, `adafruit-blinka`, `adafruit-circuitpython-neopixel`, `adafruit-circuitpython-mpr121`, `pydbus`, etc.).
* **Thread/process safety**
  LED programs run in separate processes; motors in threads. You already use locks and stop events—good. Ensure **clean shutdown** handlers (SIGTERM) to stop motors and release NeoPixel.
* **Secrets in repo**
  `update_manager.py` shows a placeholder personal access token snippet. Keep tokens out of source and read from env vars.
* **JSON schema drift**
  You already “patch defaults” from `check_*.json`. Consider formalizing with a small schema validator to catch typos early.

---

# TL;DR (what each top file does)

* **`main.py`** – boot supervisor; wires everything.
* **`webserver_microdot.py`** – REST/WebSocket control panel.
* **`led_programmliste.py`** – picks & runs LED effects; auto-dimming.
* **`led_programms/*.py`** – individual LED animations.
* **`control_motor_interpreter.py`** – high-level motor brain.
* **`tilt_motor_control.py` / `rotation_motor_control.py`** – low-level step/dir drivers.
* **`control_ladechip.py`** – MAX17261 battery gauge driver + background SOC watcher.
* **`control_compass.py`, `Kompass_Accelerometer.py`** – compass/IMU I²C reads.
* **`control_gps.py`** – GPS reader.
* **`control_sun_tracking.py`** – compute sun heading and point lamp.
* **`SW_Button.py`, `control_top_button.py`, `control_bottom_button.py`** – physical/capacitive input handlers.
* **`WIFI_Manager.py`, `Bluetooth_LE.py`** – connectivity utilities.
* **`timesync.py`** – NTP on and log.
* **`set_hostname.py` et al.** – set/normalize device hostname.
* **`current_*.json`, `check_*.json`** – runtime state and default schemas.
* **`update_manager.py`** – scheduled OTA update puller.
* **`old/`** – historical/ESP32 experiments (not used at runtime).


