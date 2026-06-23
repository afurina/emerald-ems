# emerald-ems
From Smart Meter Mandate to MQTT: Reviving a Retired Pi 3 as a BLE-to-Home-Assistant Bridge for the Emerald Electricity Advisor
Like a lot of Australians, I didn't really choose to get a smart meter — I was "opted in" as part of the government-mandated rollout, and along with it came an Emerald Electricity Advisor (EMS): a little BLE-connected energy monitor clipped onto the meter, quietly counting pulses, with zero documentation and zero interest in talking to anything outside its own phone app.

When I first set up Home Assistant, there was no integration for this thing. Not official, not community, nothing. It just sat there, blinking, while every other appliance in the house got to join the dashboard party.

Fast forward a few weeks, and I stumbled on a brilliant bit of work by WeekendWarrior1, who'd used AI assistance to reverse-engineer the Emerald's BLE protocol and get it talking to an ESP32 over Arduino/ESPHome-style firmware. Genuinely excellent groundwork — command headers, GATT characteristics, the packed timestamp format, all of it. Full credit, this project would not have happened without that reverse-engineering already being done.

But my situation had an extra wrinkle: my meter box is nowhere near my Home Assistant install, well outside any sensible BLE range, and outside WiFi range too. The "obvious" fix is an ESP32 running as an ESPHome Bluetooth Proxy near the meter, forwarding everything back over WiFi. Except this device doesn't really cooperate with that model.

Why an ESP32 BLE Proxy wasn't the right tool here

ESPHome's Bluetooth Proxy is fantastic for what it's designed for: passively hoovering up advertisements from simple BLE sensors (think Xiaomi/Govee-style devices) and occasionally doing simple active reads. The Emerald Advisor is a much fussier guest:


It demands a full authenticated pairing handshake — IO capability negotiation, MITM protection, a 6-digit passkey, the works (ESP_LE_AUTH_REQ_SC_MITM_BOND in the original firmware). That's the BLE Security Manager doing real cryptographic bonding, not just "connect and read."
It needs that bond to persist indefinitely, so future reconnects don't re-trigger the pairing dance.
It uses a custom vendor GATT protocol with write-without-response commands and notification subscriptions that Home Assistant's core Bluetooth integration has no built-in concept of — a proxy just forwards bytes, it doesn't understand "Emerald-speak."


Getting all of that right on a microcontroller means hand-rolling Security Manager callbacks in C/C++, flashing firmware for every iteration, and debugging over a serial console — exactly what the original ESP32 project had to do, line by line.

A Raspberry Pi, on the other hand, comes with this already solved. Its onboard Broadcom BLE chipset talks to a full Linux Bluetooth stack (BlueZ), which already implements the entire Security Manager Protocol, handles passkey-based bonding, and persists the bond keys to disk automatically — no firmware required at all. Pairing becomes a single bluetoothctl pair command, done once, ever. From there, a normal Python script using bleak (a high-level async BLE library that talks to BlueZ over D-Bus) can subscribe to notifications and write commands exactly like a phone app would — except the "phone app" is now a script I fully control, running on a full Linux box with proper logging, systemd supervision, and a real filesystem for persisting state. No firmware flashing, no recompiling, just edit-and-restart.

And here's the bit I really like: I had a Raspberry Pi 3 doing nothing. Retired after upgrading to a Pi 4 a while back, sitting in a drawer. It turns out a "deprecated" Pi 3 has exactly the hardware this job needs — onboard BLE radio, enough grunt to run Python comfortably, an SD card for storage — and costs nothing, because it already existed. If your meter box is far from your HA instance too, this is a genuinely good excuse to dig that old Pi out of the drawer instead of buying new hardware.

Architecture

Emerald Electricity Advisor (BLE device, vendor GATT protocol)
        |
        | Bluetooth LE (paired + bonded via BlueZ)
        v
Raspberry Pi 3 — Raspberry Pi OS Lite
  - Python script (bleak) decodes the protocol, tracks running totals
  - systemd service for resilience (auto-restart, survives reboots)
  - watchdog timer that recovers from BLE adapter lockups automatically
        |
        | MQTT (retained messages + Last Will, so HA restarts don't
        | strand entities on "Unavailable")
        v
MQTT Broker
        |
        v
Home Assistant — MQTT sensors + a derived "yesterday" template sensor

Because the bridge only needs power and a network connection, it can sit physically wherever the meter is, and just needs a path back to your MQTT broker (local network, VPN, Tailscale, whatever you've already got).

Hardware


A Raspberry Pi 3 (B/B+) — perfect use for one gathering dust
A microSD card (8GB+ is plenty)
Power supply
That's it. No proxy hardware, no extra radios, no custom PCB.


The build, condensed

(Full step-by-step — OS flashing, BlueZ pairing, systemd units, the works — is in the companion repo linked below. Here's the shape of it.)

1. Pair once, using BlueZ directly:

bashbluetoothctl
power on
agent on
default-agent
scan on
# find your device's address, then:
scan off
pair AA:BB:CC:DD:EE:FF
# enter the passkey when prompted
trust AA:BB:CC:DD:EE:FF

That's the entire "authentication" problem solved — permanently, at the OS level.

2. The bridge script (ems.py) — connects, subscribes to the live telemetry characteristic, decodes the packed frame format, and publishes to MQTT:

pythonimport asyncio
import json
import logging
import os
import time
from datetime import date
from bleak import BleakClient
import paho.mqtt.client as mqtt

EMS_ADDRESS = "AA:BB:CC:DD:EE:FF"          # <-- your device's BLE address

MQTT_BROKER = "YOUR_MQTT_BROKER_IP"
MQTT_PORT = 1883
MQTT_USER = "YOUR_MQTT_USERNAME"
MQTT_PASS = "YOUR_MQTT_PASSWORD"

TOPIC_BASE = "home/ems"

NOTIFY_CHAR_UUID   = "00002b10-0000-1000-8000-00805f9b34fb"  # data stream (notify)
WRITE_CHAR_UUID    = "00002b11-0000-1000-8000-00805f9b34fb"  # send commands here
BATTERY_CHAR_UUID  = "00002a19-0000-1000-8000-00805f9b34fb"  # standard BLE battery level

PULSES_PER_KWH = 1000  # confirmed against the original firmware's own constant

STATE_FILE = "/home/pi/ems/energy_state.json"
HEARTBEAT_FILE = "/home/pi/ems/heartbeat"

ENABLE_AUTO_UPLOAD_CMD = bytes([0x00, 0x01, 0x02, 0x0b, 0x01, 0x01])

logging.basicConfig(level=logging.INFO)


def load_state():
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE) as f:
            return json.load(f)
    return {"total_pulses": 0, "today_pulses": 0, "day": str(date.today())}


def save_state(state):
    with open(STATE_FILE, "w") as f:
        json.dump(state, f)


def touch_heartbeat():
    try:
        with open(HEARTBEAT_FILE, "w") as f:
            f.write(str(int(time.time())))
    except Exception as e:
        logging.warning(f"Could not write heartbeat file: {e}")


state = load_state()
mqtt_client = None


def mqtt_connect():
    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    client.username_pw_set(MQTT_USER, MQTT_PASS)
    # Last Will: if this script dies uncleanly, the broker marks us offline for us
    client.will_set(f"{TOPIC_BASE}/status", "reconnecting", retain=True)
    client.connect(MQTT_BROKER, MQTT_PORT, 60)
    client.loop_start()
    return client


def handle_power_notification(sender, data):
    global state
    touch_heartbeat()
    header = data[0:5].hex()

    # return30sPowerConsumptionCmd = 0001020a06
    # [5 bytes header][4 bytes packed timestamp][2 bytes pulses-in-interval]
    if header == "0001020a06" and len(data) >= 11:
        interval_pulses = (data[9] << 8) | data[10]
        power_w = interval_pulses * 120000 / PULSES_PER_KWH

        today_str = str(date.today())
        if state["day"] != today_str:
            state["day"] = today_str
            state["today_pulses"] = 0

        state["total_pulses"] += interval_pulses
        state["today_pulses"] += interval_pulses
        save_state(state)

        today_kwh = state["today_pulses"] / PULSES_PER_KWH
        total_kwh = state["total_pulses"] / PULSES_PER_KWH

        mqtt_client.publish(f"{TOPIC_BASE}/power", round(power_w, 1), retain=True)
        mqtt_client.publish(f"{TOPIC_BASE}/pulses", state["total_pulses"], retain=True)
        mqtt_client.publish(f"{TOPIC_BASE}/energy_today", round(today_kwh, 3), retain=True)
        mqtt_client.publish(f"{TOPIC_BASE}/energy_total", round(total_kwh, 3), retain=True)
        mqtt_client.publish(f"{TOPIC_BASE}/status", "online", retain=True)


def handle_battery_notification(sender, data):
    touch_heartbeat()
    mqtt_client.publish(f"{TOPIC_BASE}/battery", data[0], retain=True)


async def run():
    global mqtt_client
    mqtt_client = mqtt_connect()
    mqtt_client.publish(f"{TOPIC_BASE}/status", "starting", retain=True)

    while True:
        try:
            async with BleakClient(EMS_ADDRESS, timeout=30.0) as client:
                mqtt_client.publish(f"{TOPIC_BASE}/status", "online", retain=True)

                try:
                    await client.start_notify(BATTERY_CHAR_UUID, handle_battery_notification)
                    batt = await client.read_gatt_char(BATTERY_CHAR_UUID)
                    handle_battery_notification(None, batt)
                except Exception as e:
                    logging.warning(f"Battery read failed: {e}")

                await client.start_notify(NOTIFY_CHAR_UUID, handle_power_notification)
                await client.write_gatt_char(WRITE_CHAR_UUID, ENABLE_AUTO_UPLOAD_CMD, response=False)

                while True:
                    await asyncio.sleep(10)

        except Exception as e:
            logging.error(f"Error: {e}")
            mqtt_client.publish(f"{TOPIC_BASE}/status", "reconnecting", retain=True)
            await asyncio.sleep(5)


asyncio.run(run())

3. Run it as a systemd service so it survives reboots and crashes (/etc/systemd/system/ems.service):

ini[Unit]
Description=EMS BLE to MQTT bridge
After=bluetooth.target network-online.target
Wants=bluetooth.target network-online.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/ems
ExecStart=/home/pi/ems/venv/bin/python /home/pi/ems/ems.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

4. Home Assistant side — plain MQTT sensors, nothing exotic:

yamlmqtt:
  sensor:
    - name: "Power"
      state_topic: "home/ems/power"
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement

    - name: "Energy"
      state_topic: "home/ems/energy_total"
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing   # plugs straight into the Energy Dashboard

Two hard-won lessons worth sharing

1. The Pi 3's onboard Bluetooth adapter can silently lock up at the BlueZ level after extended use — the process keeps running, but data just stops. A small watchdog (a script run every minute via a systemd timer) checks a "heartbeat" file the main script touches on every BLE notification, and if it's gone stale, runs systemctl restart bluetooth && systemctl restart ems.service. Fully self-healing, no SSH required.

2. Don't forget retain=True on your MQTT publishes. Without it, restarting Home Assistant (not even the Pi — just HA) left every entity stuck on "Unavailable," because the availability topic was only ever published once at connection time, and HA never got it again after resubscribing. Retained messages plus a proper MQTT Last Will fixed it completely.

Scaling this up

Because the whole thing is just "a spare Pi 3 + a Python script + MQTT," it turned out to be trivially repeatable. I've since set up the same bridge at other family members' properties, each with their own bonded EMS device and their own Pi, publishing back over MQTT — letting me keep an eye on energy use across the family without needing a separate custom integration per site.

Credits


WeekendWarrior1 — for the original reverse-engineering work on the Emerald protocol and the ESP32/Arduino proof of concept this was built on top of.
The BlueZ and bleak maintainers, for making "just pair the thing properly" so much less painful on Linux than on a microcontroller.
