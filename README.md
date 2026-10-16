# 🏠 Smart Home Automation using ESP8266 & Sinric Pro

Control a home appliance with your **voice** (Alexa / Google Home) or from the **Sinric Pro app**, while the **physical wall switch keeps working** and stays in sync with the app. Built on an ESP8266 NodeMCU and a relay module.

---

## ✨ Features

- **Voice control** through Alexa or Google Home (via the Sinric Pro skill)
- **App control** from anywhere with internet, using the Sinric Pro app
- **Manual switch still works**: toggling it flips the relay *and* pushes the new state to the cloud, so the app never shows a stale state
- **State restore**: the last known device state is restored from Sinric Pro on boot
- **Wi-Fi status LED** on D0 turns on once the board is connected
- **Debounced input** (250 ms) for toggle switches or push buttons
- **Easy to extend**: devices live in one config map, so adding a second or third relay takes a few lines

---

## 🧰 Hardware

| Part | Qty | Notes |
|------|-----|-------|
| ESP8266 NodeMCU board | 1 | 2.4 GHz Wi-Fi only |
| 1-channel 5 V relay module | 1 | Driven active-LOW by this sketch |
| Toggle switch or push button | 1 | Wired between SD3 and GND |
| Appliance (lamp, fan, etc.) | 1 | Switched through the relay's COM / NO |
| 5 V power (USB or isolated AC-DC module) | 1 | Also supplies the relay module |
| Enclosure, wires, connectors | - | Mains-rated where applicable |

---

## 🧠 How it works

![System architecture](docs/architecture.svg)

1. You say *"Alexa, turn on the light"* or tap the Sinric Pro app.
2. Sinric Pro sends the command to the ESP8266 over a secure WebSocket.
3. The `onPowerState()` callback drives the relay.
4. If someone uses the manual switch instead, the ESP8266 toggles the relay and sends a **state event** back to Sinric Pro, which keeps the app and voice assistants correct.

![Program flow](docs/flowchart.svg)

---

## 🔌 Wiring

![Wiring diagram](docs/wiring.svg)

| Function | NodeMCU pin | GPIO |
|----------|-------------|------|
| Relay IN | D1 | GPIO5 |
| Manual switch (other side to GND) | SD3 | GPIO10 |
| Wi-Fi status LED | D0 | GPIO16 |
| Relay VCC / GND | VIN (5 V) / GND | n/a |

> ⚠️ **Mains safety:** switch only the **LIVE** wire through the relay's COM / NO terminals, keep high-voltage wiring in a proper enclosure, and have it checked by someone qualified. Never work on mains while it is powered.

<!-- Add your own build photos: ![Prototype](docs/prototype.jpg) -->

---

## 🚀 Getting started

### 1. Set up Sinric Pro

1. Create an account at [sinric.pro](https://sinric.pro) and add a new **Switch** device.
2. Copy the **Device ID**, then open *Credentials* and copy the **App Key** and **App Secret**.
3. Link the **Sinric Pro** skill in the Alexa app or the **Sinric Pro** action in Google Home so the device shows up in your assistant.

### 2. Set up the Arduino IDE

1. Install the **ESP8266 board package** (Boards Manager) and select your NodeMCU board.
2. Install the **SinricPro** library from the Library Manager. If the IDE does not pull them in automatically, also install **ArduinoJson** and **WebSockets**.

### 3. Configure and upload

Edit the placeholders at the top of the sketch:

```cpp
#define WIFI_SSID    "YOUR_WIFI_NAME"
#define WIFI_PASS    "YOUR_WIFI_PASSWORD"
#define APP_KEY      "YOUR_SINRIC_APP_KEY"
#define APP_SECRET   "YOUR_SINRIC_APP_SECRET"
#define device_ID_1  "YOUR_DEVICE_ID"
```

Upload, open the Serial Monitor at **9600 baud**, and wait for `connected!`. The Wi-Fi LED turns on, and the device shows as online in the Sinric Pro dashboard.

---

## ⚙️ Configuration

| Setting | Default | Meaning |
|---------|---------|---------|
| `WIFI_SSID` / `WIFI_PASS` | placeholder | Your 2.4 GHz Wi-Fi credentials |
| `APP_KEY` / `APP_SECRET` | placeholder | From the Sinric Pro *Credentials* page |
| `device_ID_1` | placeholder | Device ID of your Switch device |
| `RelayPin1` | `5` (D1) | Relay control pin |
| `SwitchPin1` | `10` (SD3) | Manual switch input (internal pull-up) |
| `wifiLed` | `16` (D0) | Wi-Fi status LED |
| `TACTILE_BUTTON` | commented out | Uncomment for push buttons; leave commented for toggle switches |
| `DEBOUNCE_TIME` | `250` ms | Input debounce time |
| `BAUD_RATE` | `9600` | Serial Monitor speed |
| `ENABLE_DEBUG` | off | Define it to enable Sinric Pro debug output |

### Adding more devices

Every setup function loops over one `devices` map, so a second relay is a small change:

```cpp
#define device_ID_2   "YOUR_SECOND_DEVICE_ID"
#define RelayPin2     4    // D2
#define SwitchPin2    14   // D5

std::map<String, deviceConfig_t> devices = {
    {device_ID_1, { RelayPin1, SwitchPin1 }},
    {device_ID_2, { RelayPin2, SwitchPin2 }}
};
```

Add the matching Switch device in the Sinric Pro dashboard and you are done.

---

## 🔐 Keep your credentials out of Git

- Never commit real Wi-Fi passwords, App Key, App Secret or Device IDs. Keep placeholders in the repo and put real values only in your local copy.
- If real values were ever pushed, treat them as exposed: **regenerate the App Secret in Sinric Pro and change the Wi-Fi password**. Deleting them in a later commit does not remove them from Git history.

---

## 🔧 Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Stuck on `Connecting....` | Wrong SSID/password, or a 5 GHz-only network (the ESP8266 supports 2.4 GHz only) |
| Device shows offline in Sinric Pro | Wrong App Key, App Secret or Device ID, or no internet access |
| Relay behaves inverted | Your module is active-HIGH, so flip the logic in `setupRelays()`, `onPowerState()` and `handleFlipSwitches()` |
| Switch input misreads or the board boots erratically | GPIO10 (SD3) is a flash pin on some modules. Try *Tools → Flash Mode → DIO*, or move the switch to a free GPIO such as D5 |
| App state is wrong after using the manual switch | Make sure the device is online, since the state event is only sent while connected |
| Assistant says "device not responding" | Re-link the Sinric Pro skill and run device discovery again |

---

## 📁 Repository structure

```
smart-home-automation-by-sinric-pro-/
├── <your_sketch>.ino
├── docs/
│   ├── architecture.svg / architecture.png
│   ├── wiring.svg / wiring.png
│   └── flowchart.svg / flowchart.png
└── README.md
```

---

## 🗺️ Roadmap

- Multi-channel version (4-relay board)
- Over-the-air (OTA) firmware updates
- Power monitoring and usage reports
- Move credentials into a separate `secrets.h` that is git-ignored

---

## prototype
![image alt](https://github.com/Piyush9301/smart-home-automation-by-sinric-pro-/blob/1056091f998048a1b8b3c38c964dc394f75a0988/IMG20260617003857.jpg)


## 📄 License

MIT. Free to use, modify, and share.

## 👤 Author

**Piyush Bopche** ([@Piyush9301](https://github.com/Piyush9301)), Electronics & Communication Engineering, LNCT Bhopal
