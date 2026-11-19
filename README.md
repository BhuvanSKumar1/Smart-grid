# AI-Powered Smart Grid

A home smart-grid prototype that monitors appliance loads over MQTT, protects the circuit automatically, sends alerts to Telegram, and shows everything on a live Streamlit dashboard.

It uses **hardware-in-the-loop (HIL)** simulation: a real ESP32 node and a Python "virtual house" publish telemetry on the same MQTT topics, so the backend treats real and simulated loads the same way.

## Features

- **Virtual house simulator**: models a ceiling fan, an AC with a compressor cycle, a solar panel with a day curve and cloud dips, and a washing machine with a 30-minute cycle
- **Overload protection**: trips only the faulty load when it exceeds its power limit
- **Overvoltage load shedding**: above 250 V, cuts non-essential loads (washer, then AC) and keeps the fan running. Restores them below 240 V
- **Telegram alerts**: sends every protection event to your phone
- **ESP32 LCD feed**: publishes live totals, per-load ON/OFF status, and short alerts for a 16x2 LCD
- **Telemetry logging**: saves every reading to `telemetry_log.csv` for analysis and ML training
- **Live dashboards**: two Streamlit dashboards with load controls and demo fault buttons

## Architecture

```
 ESP32 node (real) ─┐                         ┌─> Telegram alerts
                    ├──> Mosquitto MQTT ──> backend.py ──┼─> telemetry_log.csv
 virtual_house.py ──┘     (1883 / 9001 ws)     │         └─> home1/summary, home1/alerts (LCD)
         ▲                                     │
         └──── commands (ON/OFF, faults) <─────┴──── dashboard.py / dashboard_v2.py
                                                     (reads telemetry_log.csv)
```

## Project Structure

| File | Purpose |
|------|---------|
| `virtual_house.py` | Simulates four household loads. Publishes telemetry every 2 s. 1 real second = 1 simulated minute, and the simulated day starts at 08:00 |
| `backend.py` | Protection logic: overload trips, overvoltage shedding, Telegram alerts, LCD summary, CSV logging |
| `dashboard.py` | Basic Streamlit live dashboard |
| `dashboard_v2.py` | "Mission Control" dashboard: Plotly charts, voltage view, event log, status badge |
| `test_telegram.py` | Standalone check that your Telegram bot token and chat ID work |
| `test_ws.html` | Browser test page for the MQTT WebSocket listener (`ws://localhost:9001/mqtt`) |
| `mosquitto.conf` | Broker config: TCP on 1883 and WebSockets on 9001, anonymous access |
| `mosquitto-ws-bridge.conf` | Optional second broker that serves WebSockets on 9001 and bridges all topics to a main broker on 1883 |
| `telemetry_log.csv` | Telemetry log written by the backend (`ts,node,load,v,i,p,state`) |
| `smart-grid-dashboard` | Submodule reference to a separate dashboard project. Its contents are not in this repo |

## Requirements

- Python 3.9+
- [Mosquitto](https://mosquitto.org/download/) MQTT broker
- Python packages:

```bash
pip install "paho-mqtt>=2.0" streamlit streamlit-autorefresh pandas plotly
```

## Configuration

These settings are constants at the top of the scripts:

| Setting | File | Default |
|---------|------|---------|
| `BROKER`, `PORT` | all scripts | `localhost`, `1883` |
| `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_IDS` | `backend.py` | your bot token from @BotFather and your chat ID from @userinfobot |
| `POWER_LIMITS` | `backend.py` | fan 150 W, AC 2000 W, washer 2500 W |
| `OVERVOLT_TRIP` / `OVERVOLT_CLEAR` | `backend.py` | 250 V / 240 V |
| `SHED_ORDER` | `backend.py` | `["washer", "ac"]` (the fan counts as essential) |
| `SPEED`, `PUBLISH_EVERY` | `virtual_house.py` | 60x, 2 s |

> Do not commit a real Telegram bot token. If a token has been pushed, revoke it in @BotFather and keep the new one outside version control.

## Running

Start each component in its own terminal, in this order:

```bash
# 1. MQTT broker
mosquitto -c mosquitto.conf -v

# 2. Virtual house simulator
python virtual_house.py

# 3. Backend (protection + alerts + logging)
python backend.py

# 4. Dashboard (pick one)
streamlit run dashboard_v2.py
streamlit run dashboard.py
```

Streamlit serves the dashboard at `http://localhost:8501`. To check Telegram on its own, run `python test_telegram.py`.

## Demo Scenarios

Run these from the dashboard's **Demo Controls**, or publish to the topics directly with `mosquitto_pub`:

| Scenario | Topic / payload | Expected result |
|----------|-----------------|-----------------|
| Fan fault | `home1/real/fault/inject` = `overload_fan` | Fan power rises above 150 W. The backend trips only the fan and sends an alert |
| Voltage swell | `home1/sim/voltage` = `262` | The washer and AC are shed and the fan stays on |
| Voltage normal | `home1/sim/voltage` = `230` | Shed loads are restored automatically |
| Toggle a load | `home1/virtual/<load>/cmd/manual` = `ON` / `OFF` | The load switches. Turning a load back ON re-arms its protection |
| Full reset | `home1/reset` | Clears all trips and shedding |

On the real ESP32 node, pressing its button publishes the fan-fault message.

## MQTT Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `home1/<node>/<load>/telemetry` | node → backend | JSON `{"ts", "v", "i", "p", "state"}`. `node` is `real` or `virtual`. Solar `p` is negative (generation) |
| `home1/virtual/<load>/cmd/manual` | user → house | `ON` / `OFF` |
| `home1/virtual/<load>/cmd/auto` | backend → house | `ON` / `OFF` (protection actions) |
| `home1/virtual/<load>/cmd` | dashboard → house | `ON` / `OFF` (legacy topic) |
| `home1/real/fault/inject` | button/dashboard → house | `overload_fan` |
| `home1/sim/voltage` | dashboard → house | Grid voltage in volts, e.g. `262` |
| `home1/reset` | any → backend | Resets protection state |
| `home1/summary` | backend → LCD | `P:<total W>\|L:<active loads>\|S:FAN:ON\|AC:ON\|WAS:OFF` |
| `home1/alerts` | backend → LCD | Short alert text |
| `home1/virtual/status` | house | `online` / `offline` (retained, last will) |
