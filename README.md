# APRS Weather Station Setup Documentation

This repository documents the complete setup of an APRS weather station using multiple Raspberry Pi devices. The system integrates data from an AcuRite 5-in-1 outdoor weather station and an indoor BMP280/BME280 temperature sensor, processes it through Node-RED, stores it in InfluxDB, visualizes it in Grafana, and transmits position + weather information via Direwolf over APRS radio (both RF and APRS-IS). Additional weather reporters can be added by copying the code for the MainWX Pi and changing the MQTT topic parameters to show the location of the sensor; then adding code to the NodeRed script on the WXStation Pi to subscribe to that topic and transfer the data to Influx.

## Overview

**Devices & Roles**

| Device       | IP address       | Primary Role                                                                 |
|--------------|------------------|------------------------------------------------------------------------------|
| WXStationPi  | 192.168.50.63    | rtl_433 → MQTT (outdoor sensors), Node-RED processing, InfluxDB write, APRS beacon text generation |
| Weather Pi   | 192.168.50.60    | Runs Direwolf – reads beacon text file and transmits APRS packets (RF + IGate) |
| MainWX Pi    | 192.168.50.153   | Reads indoor temperature (BME280) via Node-RED → publishes to MQTT            |

**Data Flow**

1. AcuRite 5-in-1 → 433 MHz → rtl_433 → MQTT stream (WXStationPi)
2. BME280 indoor sensor → Node-RED → MQTT stream (MainWX Pi)
3. Node-RED (WXStationPi) subscribes to both streams, formats APRS weather text, writes to file
4. Direwolf (Weather Pi) reads the text file and transmits as APRS beacon (RF + internet)
5. InfluxDB stores all weather values
6. Grafana visualizes the data from InfluxDB

---

## Hardware

- **WXStationPi** & **Weather Pi**: Raspberry Pi 3 or 4 recommended
- RTL-SDR dongle + antenna (WXStationPi)
- AcuRite 5-in-1 weather station (outdoor)
- BME280/BMP280 I²C sensor (indoor – MainWX Pi)
- USB sound card (C-Media CM108/CM119) + radio transceiver for APRS RF
- Stable local network (all devices on same subnet)

---

## Software Components

- Raspberry Pi OS (Bookworm recommended)
- rtl_433 (433 MHz receiver/decoder)
- Mosquitto MQTT broker
- Node-RED (data processing & formatting)
- InfluxDB (time-series storage)
- Grafana (visualization)
- Direwolf (APRS TNC & beacon transmitter)

---

## 1. WXStationPi – rtl_433 + MQTT (Outdoor Sensors)

### Install rtl_433

```bash
sudo apt update
sudo apt install rtl-433 -y
```

### Run rtl_433 as Service

Create `/etc/systemd/system/rtl433.service`:

```ini
[Unit]
Description=rtl_433 AcuRite 5-in-1 to MQTT
After=network.target

[Service]
ExecStart=/usr/bin/rtl_433 -R 40 -f 433920000 -F json -F mqtt://192.168.50.63:1883,retain=0,devices=weather/devices/Acurite-5n1/A/3529
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable rtl433.service
sudo systemctl start rtl433.service
```

**Verify**:
```bash
mosquitto_sub -h 192.168.50.63 -t "weather/#" -v
```

---

## 2. MainWX Pi – Indoor Temperature via Node-RED

### Install Node-RED on Pi Zero / Any Pi

```bash
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
sudo systemctl enable nodered.service
sudo systemctl start nodered.service
```

Access: `http://192.168.50.153:1880`

### Indoor Temperature Flow (Node-RED)

Import this flow:

```json
[
  {
    "id": "temp-indoor-to-mqtt",
    "type": "tab",
    "label": "Indoor Temp → MQTT",
    "disabled": false,
    "info": ""
  },
  {
    "id": "bme280-sensor",
    "type": "inject",
    "z": "temp-indoor-to-mqtt",
    "name": "Every 60 s",
    "props": [
      {
        "p": "payload"
      }
    ],
    "repeat": "60",
    "crontab": "",
    "once": false,
    "onceDelay": 0.1,
    "topic": "",
    "payload": "",
    "payloadType": "date",
    "x": 140,
    "y": 100,
    "wires": [
      [
        "read-bme280"
      ]
    ]
  },
  {
    "id": "read-bme280",
    "type": "function",
    "z": "temp-indoor-to-mqtt",
    "name": "Read BME280 & convert to °F",
    "func": "const bus = global.get('bme280_bus');\nif (!bus) {\n    node.error(\"BME280 bus not initialized\");\n    return null;\n}\n\ntry {\n    const data = bme280.sample(bus, 0x76);\n    const tempC = data.temperature;\n    const tempF = (tempC * 9/5) + 32;\n    \n    msg.payload = tempF.toFixed(2);\n    return msg;\n} catch(err) {\n    node.error(\"BME280 read error: \" + err);\n    return null;\n}",
    "outputs": 1,
    "noerr": 0,
    "initialize": "",
    "finalize": "",
    "libs": [],
    "x": 360,
    "y": 100,
    "wires": [
      [
        "publish-to-mqtt"
      ]
    ]
  },
  {
    "id": "publish-to-mqtt",
    "type": "mqtt out",
    "z": "temp-indoor-to-mqtt",
    "name": "Indoor Temp → MQTT",
    "topic": "weather/indoor/temperature_F",
    "qos": "0",
    "retain": "false",
    "respTopic": "",
    "contentType": "",
    "userProps": "",
    "correl": "",
    "expiry": "",
    "broker": "clubhouse-mqtt",
    "x": 610,
    "y": 100,
    "wires": []
  },
  {
    "id": "clubhouse-mqtt",
    "type": "mqtt-broker",
    "name": "Clubhouse MQTT",
    "broker": "192.168.50.63",
    "port": "1883",
    "clientid": "MainWX-IndoorTemp",
    "autoConnect": true,
    "usetls": false,
    "protocolVersion": "4",
    "keepalive": "60",
    "cleansession": true,
    "birthTopic": "",
    "birthQos": "0",
    "birthPayload": "",
    "birthMsg": {},
    "closeTopic": "",
    "closeQos": "0",
    "closePayload": "",
    "closeMsg": {},
    "willTopic": "",
    "willQos": "0",
    "willPayload": "",
    "willMsg": {},
    "userProps": "",
    "sessionExpiry": ""
  }
]
```

**Notes**:
- Uses the same MQTT broker as the outdoor station.
- Publishes every 60 seconds.
- Requires `node-red-contrib-bme280` (install via Manage Palette).

---

## 3. WXStationPi – Node-RED Processing & APRS Beacon Text

Node-RED subscribes to both outdoor (rtl_433) and indoor MQTT topics, formats the APRS weather string, and writes it to the file that Direwolf reads.

### Significant Direwolf Beacon Lines (from direwolf.conf)

```conf
PBEACON delay=1 every=10:00 lat=39^43.02N long=75^12.60W symbol="wx" comment="GCARC Clubhouse Mullica Hill NJ" VIA=WIDE1-1,WIDE2-1

CBEACON delay=1:00 every=10:00 LAT=39^43.02N LONG=75^12.60W SYMBOL=wx INFO="/home/gcarc/weather.txt" VIA=WIDE1-1,WIDE2-1
```

### Node-RED Flow Summary

- 5 MQTT-in nodes for outdoor sensors
- 1 MQTT-in node for indoor temperature (`weather/indoor/temperature_F`)
- Change nodes to store values in flow context
- Rain delta calculation & hourly reset
- Main formatting function produces APRS weather string (wind/dir, gust, temp, rain, humidity, placeholders)
- Writes result to `/home/gcarc/weather.txt`
- Optional dashboard gauges/text

---

## 4. Weather Pi – Direwolf APRS Transmission

Direwolf reads `/home/gcarc/weather.txt` every 10 minutes and transmits the full APRS packet (position + weather) over RF and APRS-IS.

### Critical Direwolf Lines

```conf
ADEVICE plughw:1,0
ACHANNELS 1
PTT CM108

MYCALL W2MMD-13
MODEM 1200

IGSERVER noam.aprs2.net
IGLOGIN W2MMD-13 <your-passcode>

PBEACON delay=1 every=10:00 lat=39^43.02N long=75^12.60W symbol="wx" comment="GCARC Clubhouse Mullica Hill NJ" VIA=WIDE1-1,WIDE2-1

CBEACON delay=1:00 every=10:00 LAT=39^43.02N LONG=75^12.60W SYMBOL=wx INFO="/home/gcarc/weather.txt" VIA=WIDE1-1,WIDE2-1
```

---

## 5. InfluxDB & Grafana

### InfluxDB
- Server: 192.168.50.37:8086
- Database: `wx`
- Measurement: `WXstation`
- Fields: `OutdoorTemp`, `Humidity`, `ClubhouseTemp`, etc.

### Grafana
- Access: `http://192.168.50.63:3000`
- Data source: InfluxDB (URL: `http://192.168.50.37:8086`, database `wx`)
- Example query (Clubhouse temperature):
  ```sql
  SELECT "ClubhouseTemp" FROM "WXstation" WHERE $timeFilter
  ```

---

## Critical Areas & Lessons Learned

- **rtl_433 decoding** → must use `-R 40` for AcuRite 5-in-1; test with manual bucket tipping
- **MQTT topic consistency** → all devices must use same broker & topic pattern
- **Node-RED file path** → use absolute path `/home/gcarc/weather.txt`
- **Direwolf `CBEACON`** → `INFO=`, not `info=` or `infocmd=`; always include `LAT`/`LONG`/`SYMBOL`
- **PTT CM108** → requires correct audio device (`plughw:X,0`) and GPIO wiring
- **Passcode** → must be valid for `W2MMD-13` (re-generate if packets drop silently)
- **Python → MQTT** → use `paho-mqtt` with proper reconnect logic

This system is now fully operational for both outdoor/indoor weather reporting via APRS and time-series storage/visualization.

For questions or expansions (pressure, soil moisture, etc.), open an issue.

73 and happy weather beaconing!  
Jon – GCARC / W2MMD-13
