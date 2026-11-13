# Deye inverter → Home Assistant over Bluetooth (ESPHome, ESP32)

🇵🇹 [Versão em português](README.pt.md)

Read a **Deye three‑phase hybrid inverter** in real time from Home Assistant, **locally**, through the Bluetooth LE interface of its **WiBLE logger stick**, using a cheap ESP32 running ESPHome.

No cloud, no Modbus wiring, no changes to the inverter. One ESP32 near the inverter, one YAML file.

Tested on a **Deye SUN‑10K‑SG05LP3‑EU‑SM2** (three‑phase, low‑voltage 48 V battery) with a **WiBLE** logger (firmware `DYDA_WiBLE_1.6.0`) and an **ESP32‑C6‑DevKitC‑1**. It has been running in production since August 2026, polling every 30 s.

---

## Why this exists

The newer Deye **WiBLE** logger sticks are cloud‑only:

- They **do not** expose Modbus TCP (port 8899) or the UDP discovery port (48899), so [ha‑solarman](https://github.com/davidrapan/ha-solarman) and similar local integrations cannot talk to them. In ha‑solarman this shows up as `'NoneType' object has no attribute 'parser'`; that is the closed port, not a configuration problem.
- No firmware downgrade opens those ports.

The stick **does** have a Bluetooth LE interface (used by the Deye app for commissioning), and it forwards raw Modbus RTU frames to the inverter. This project uses that path.

## What you get

61 sensors and one control in Home Assistant:

| Group | Entities |
|---|---|
| **Energy counters** (inverter‑maintained, ideal for the Energy dashboard) | Daily/Total production, load consumption, energy bought/sold, battery charge/discharge |
| **Battery** | SOC, voltage, current, power (+ charging / − discharging), temperature |
| **Grid** | L1–L3 voltage, frequency, internal CT power, external CT L1–L3 + total, grid L1–L3 + total power |
| **Inverter output** | L1–L3 voltage and power, total power, frequency |
| **Load** | L1–L3 voltage and power, total load, UPS power |
| **PV** | PV1–PV4 power, voltage and current, total PV power |
| **Temperatures** | DC (heatsink) and AC temperature |
| **Control** | `BLE Reconnect Interval` (number, see below) |

## Hardware

- Any **ESP32 with Bluetooth LE** supported by ESPHome. The config targets an `esp32-c6-devkitc-1` (8 MB flash, ESP‑IDF). For another board, change the `esp32:` block.
- Place it within good Bluetooth range of the logger stick (a couple of metres; metal enclosures hurt).
- USB power.

## Protocol

From community reverse engineering of the Deye app's Bluetooth traffic:

| | |
|---|---|
| GATT service | `0x0922` |
| Write characteristic | `0xFEC7` |
| Notify characteristic | `0xFED8` |
| Request | `AT+INVDATA=8,<Modbus RTU frame in hex + CRC16>\r\n` |
| Response | `+ok=<Modbus RTU response in hex, incl. CRC>\r\n` |

The ESP builds a standard Modbus RTU *Read Holding Registers* frame (`01 03 <start> <count>`), adds the CRC16 at runtime, wraps it in the `AT+INVDATA` command and parses the `+ok=` reply.

### Register map

This uses the **three‑phase Deye hybrid map** (the same one ha‑solarman calls `deye_p3`), **not** the single‑phase map that circulates in older shared configs. Registers are read in 9 blocks of at most 14 registers each, which keeps every reply inside the Bluetooth MTU:

| # | Start | Count | Content |
|---|---|---|---|
| 1 | `0x0202` | 12 | Battery charge/discharge, grid import/export (daily + total) |
| 2 | `0x020E` | 10 | Load consumption, PV production (daily + total) |
| 3 | `0x021C` | 2 | DC / AC temperature |
| 4 | `0x024A` | 6 | Battery: temperature, voltage, SOC, power, current |
| 5 | `0x0256` | 12 | Grid L1–L3 voltage, internal CT power, frequency |
| 6 | `0x0265` | 13 | External CT, power factor, grid L1–L3 and total power |
| 7 | `0x0273` | 12 | Inverter output L1–L3 V/I/P, total, frequency |
| 8 | `0x0280` | 14 | UPS power, load L1–L3 voltage and power, total load |
| 9 | `0x02A0` | 12 | PV1–PV4 power, voltage and current |

Notes on decoding:

- 32‑bit counters are **low word first** (Deye convention).
- Temperatures are `(raw − 1000) × 0.1 °C`.
- Power values are signed 16‑bit.
- Each reply is checked against the byte count expected for the block that was requested. A late reply that arrives after the poller has moved on is dropped, so it is never decoded with the wrong register map. This happens right after connecting.

> **Single‑phase inverter?** The protocol is the same, but the registers are not. Replace the `cmd_*` frames and the parser cases with the single‑phase map (ha‑solarman's `deye_hybrid.yaml` is a good reference).

## Installation

1. Install ESPHome (`pip install esphome`, or the ESPHome Device Builder in Home Assistant).
2. Copy `deye-sun10k-ble.yaml` and `secrets.yaml.example` into your ESPHome folder, then rename the example to `secrets.yaml` and fill it in.
3. Set `ble_mac` in the `substitutions:` block to **your** logger's Bluetooth MAC (see below).
4. Flash the first time over USB:
   ```bash
   esphome run deye-sun10k-ble.yaml
   ```
   After that, updates go over Wi‑Fi (OTA).
5. Home Assistant discovers the device. Add it with the API encryption key from `secrets.yaml`.

### Find the logger MAC

- Scan with a phone app such as **nRF Connect**. The stick advertises the `0x0922` service, so pick the device that does.
- Or temporarily add an `esp32_ble_tracker:` `on_ble_advertise` logger to the ESP and look at the logs.

## Using it in Home Assistant

### Energy dashboard

Use the **native counters** kept by the inverter. They do not drift and they survive restarts:

| Energy dashboard slot | Entity |
|---|---|
| Grid consumption | `Total Energy Bought` |
| Return to grid | `Total Energy Sold` |
| Solar production | `Total Production` |
| Battery: energy in / out | `Total Battery Charge` / `Total Battery Discharge` |

### Real‑time vs. Deye Cloud: `BLE Reconnect Interval`

The logger stick accepts **only one Bluetooth session at a time**. While the ESP holds the connection, the stick does not upload to Deye Cloud, so the Deye app and any cloud integration stop updating.

You choose with the `BLE Reconnect Interval` number entity:

| Value | Behaviour |
|---|---|
| `0` (default) | **Permanent connection.** Real‑time data every 30 s; Deye Cloud is frozen. |
| `> 0` seconds | **Read and disconnect.** The ESP reads all blocks, drops the connection and reconnects after N seconds. Deye Cloud keeps working between reads. |

Setting it back to `0` while disconnected reconnects immediately.

A watchdog forces a reconnect in permanent mode if no valid reply arrives for 90 s.

## ESP32‑C6 notes

- The logger is set to `hardware_uart: UART0`. The default on the C6 is USB‑JTAG, which does not show up on many USB‑serial adapters.
- If you monitor the serial port with your own script (pyserial), release DTR/RTS. Otherwise the chip is held in reset and looks dead.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Never connects | Wrong `ble_mac`; ESP too far from the stick; the Deye app is connected to the stick at the same time. |
| Connects, but all values are `unknown` | Single‑phase inverter or different register map (see above). |
| `Byte count … dropping frame` in the logs now and then | Expected right after connecting: a late reply is being discarded safely. |
| Deye app or cloud stopped updating | Expected with `BLE Reconnect Interval = 0`; raise it if you need the cloud. |

## Credits

The Bluetooth protocol comes from community reverse‑engineering work on the Deye app. The three‑phase register map follows the `deye_p3` definitions of [ha‑solarman](https://github.com/davidrapan/ha-solarman).

## License

MIT. See [LICENSE](LICENSE). Use at your own risk: this only **reads** registers (function `0x03`), but it talks to equipment connected to mains and batteries.
