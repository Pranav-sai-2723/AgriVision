<p align="center">
  <img src="docs/images/agrivision-rover.png" alt="AgriVision rover (AI-generated concept image)" width="720">
</p>

<h1 align="center">AgriVision</h1>
<p align="center"><b>Autonomous Crop Disease Detection and Targeted Spraying Rover</b></p>
<p align="center"><i>Hackathon prototype: October 2026</i></p>

---

## What is AgriVision?

Farmers usually find crop diseases late, by eye, and then spray the whole field. This wastes pesticide and money and harms soil and health. AgriVision is a small rover (about 10-20 cm wide) designed to drive between two crop rows, photograph the leaves, detect disease with an AI model, and spray only the infected plants.

Target crops are row crops with a clear gap between rows, such as tomato, chilli and brinjal.

> The image above is an AI-generated concept picture, not a photo of the finished rover.

## Prototype status

This repository holds the **hackathon prototype**. It is smaller than the full proposal.

| Part | Proposal | Hackathon prototype |
|---|---|---|
| Drive | 4WD, encoders, 2 motor drivers | 4 DC motors on an Arduino Uno + HW-130 shield, no encoders |
| Navigation | Ultrasonic, IMU, PID row following | Manual keyboard control (autonomy is future work) |
| Camera | One camera per side | One USB webcam on a rotating servo |
| Spraying | 2 pumps, 2 tanks, nozzles | 2 servos stand in for sprayers (no tanks or pumps yet) |
| Extra | None | Servo that lowers a soil-moisture probe |
| AI | Classifier over Wi-Fi | FastAPI server on a teammate's laptop |

### What works today

- [x] Raspberry Pi 4 set up headless over a phone hotspot (SSH)
- [x] Live camera view in a browser (MJPEG, port 8080)
- [x] Photo capture on a key press, saved with date and time
- [x] Photo upload to the AI server and a disease class plus confidence in reply
- [x] Camera servo (10-170 degrees)
- [ ] Wheels: motor directions wrong and a fumes incident; board must be inspected before powering again
- [ ] Sprayer servo 1 partly tested; sprayer servo 2 and the moisture probe not wired yet
- [ ] Disease-to-pesticide-to-sprayer lookup (auto-spray is switched off)
- [ ] Autonomy: row following and plant localization
- [ ] Real pumps and tanks

## How it works

```
USB webcam ─► Raspberry Pi 4 ──Wi-Fi──► AI server (FastAPI, laptop)
                  │                          │
                  │  ◄──── JSON reply ───────┘
                  │       predicted_class, confidence, model
                  │
                  └─ USB serial (9600 baud) ─► Arduino Uno + HW-130 shield
                                                ├─ 4 DC motors (wheels)
                                                └─ 4 servos: camera, sprayer 1,
                                                   sprayer 2, moisture probe
```

1. The rover drives to a plant and stops.
2. The webcam, turned by its servo, takes a photo of the leaves.
3. The Pi uploads the photo over Wi-Fi to the AI server.
4. The server returns the predicted disease class, a confidence percentage and the model name.
5. A lookup (still to be added) will turn the disease class into a pesticide and a sprayer, and the Pi will move that sprayer servo.

Two run modes are planned: **single pass** (stop, capture, classify and spray at each plant) and **two pass** (scout the row first, then return and spray). Single pass is simpler and more reliable in mud. A dual-rover version (one scout, one sprayer) is future work.

If you have a block diagram, add it here:

```markdown
![System block diagram](docs/images/block-diagram.png)
```

## Hardware

| Component | Role |
|---|---|
| Raspberry Pi 4 | Brain: control script, photos, upload, live view |
| USB webcam on a servo | Leaf photos (an 8 MP Pi camera is available but not used) |
| Arduino Uno + HW-130 motor shield (2x L293D) | Drives 4 DC motors and the servos |
| 4 DC gear motors on a cardboard chassis | Wheels |
| 4 servos | Camera rotation, sprayer 1, sprayer 2, moisture probe |
| 3 x 2500 mAh Li-ion cells in series, TP4056 charger modules | Motor power and charging |
| 5 V 3 A power bank | Powers the Pi for now |
| ESP32, nRF module, AMS1117-5V | Not used yet; reserved for later |

### Pin map

| Item | Connects to |
|---|---|
| Motors M1-M4 | HW-130 terminals (M1 front-left, M2 rear-left, M3 front-right, M4 rear-right) |
| Battery | EXT_PWR terminal, through a switch. **PWR jumper removed** |
| Camera servo | SER1 header (D10) |
| Sprayer servo 1 | SER2 header (D9) |
| Sprayer servo 2 | A0, plus 5 V and GND |
| Moisture probe servo | A1, plus 5 V and GND |
| Pi to Uno | USB cable, appears as `/dev/ttyUSB0` |
| Webcam | USB, appears as `/dev/video0` |

### Power and safety rules

- **Remove the PWR jumper** on the HW-130 so the battery feeds only the motors.
- A fully charged 3S pack is 12.6 V, which is too high for typical 3-6 V gear motors. **Check the motor rating, and set a buck converter to the motor voltage with a multimeter before connecting.**
- Power the Pi from a 5 V 3 A source, never from the AMS1117 (limited to 1 A).
- Give the servos their own 5 V supply at the final stage, with a common ground.
- The HW-130 needs the **Adafruit Motor Shield library (V1)**. The V2 library will not work.
- Each L293D channel handles only 0.6 A, so use one motor per channel at low PWM (150 or below).
- Charge cells one at a time in a protected charger. Never charge a series pack through a single-cell TP4056, and never leave cells charging unattended.
- **Before powering the rover again:** disconnect everything, check the board for burn marks, measure the battery voltage, and test one motor at low PWM.

## Setup

### 1. Arduino Uno

1. Install the **Adafruit Motor Shield library (V1)** in the Arduino IDE.
2. Select board **Arduino Uno**, then upload `rover.ino` before connecting the Uno to the Pi.

### 2. Raspberry Pi (headless)

1. Write **Raspberry Pi OS Lite (64-bit)** to the SD card with Raspberry Pi Imager. Under *Edit Settings* set the hostname (`agripi`), username, password, hotspot name and password, and enable SSH.
2. Power the Pi, wait 3-5 minutes, then connect from the laptop on the same hotspot:

```bash
ssh your-username@agripi.local      # or use the Pi's IP address
```

3. Install the libraries (use `apt`, not `pip`, because of the externally-managed-environment protection):

```bash
sudo apt update
sudo apt install -y python3-serial python3-opencv python3-requests python3-flask
sudo usermod -aG dialout $USER
sudo reboot
```

4. Check the connections:

```bash
ls /dev/ttyUSB* /dev/video*
python3 -c "import flask, requests, cv2, serial; print('all good')"
```

5. Open `rover.py` and set the server address to the server laptop's IPv4 address:

```python
SERVER = "http://<SERVER_IP>:8000/predict"
AUTO_SPRAY = False   # keep off until the pesticide lookup exists
```

### 3. AI server (teammate's laptop)

Run from the folder that contains `app.py`:

```powershell
uvicorn app:app --host 0.0.0.0 --port 8000
```

- It must show `0.0.0.0:8000`, not `127.0.0.1:8000` (check with `netstat -ano | findstr :8000`).
- Allow the port once in an Administrator PowerShell:

```powershell
New-NetFirewallRule -DisplayName "FastAPI 8000" -Direction Inbound -Protocol TCP -LocalPort 8000 -Action Allow -Profile Any
```

- The Pi, the laptop and the server laptop must be on the **same phone hotspot**. Campus Wi-Fi often blocks devices from talking to each other.
- Read the server laptop's IP with `ipconfig`, because it changes between networks.

### 4. Run

```bash
python3 rover.py
```

Open `http://<PI_IP>:8080/` in a browser on the laptop to see the live camera. Use the Pi's IP address, because `agripi.local` often fails in a Windows browser.

## Controls

| Key | Action |
|---|---|
| `W` `S` `A` `D` | Forward, backward, spin left, spin right (stops 0.7 s after the last command) |
| `Space` | Stop |
| `Q` / `E` / `F` | Camera servo left / right by 10 degrees / back to centre |
| `C` | Capture a photo and upload it in the background |
| `1` / `2` | Move sprayer servo 1 / 2 out and back |
| `M` / `N` | Lower / raise the moisture probe |
| `X` | Quit (restores the terminal and stops the motors) |

Safety built into the sketch: servos are limited to 10-170 degrees and move in small steps, the wheels stop before any sprayer or probe moves, and the wheels stop if no drive command arrives for 0.7 seconds.

## Server API

**`POST /predict`** (port 8000). Send the image as a form field named **`file`**.

```bash
curl -F "file=@photos/leaf_YYYYMMDD_HHMMSS.jpg" http://<SERVER_IP>:8000/predict
```

Reply:

```json
{
  "predicted_class": "<class name>",
  "confidence": 19.35,
  "model": "<model name>"
}
```

`confidence` is **already a percentage** (0-100). Do not multiply it by 100. A `405` reply to a plain GET only means the server is reachable.

## AI model

- Image classifier running on the teammate's server (MobileNet / EfficientNet-Lite class). `TODO: confirm the model, dataset and class list.`
- Planned data: a public leaf-disease dataset such as PlantVillage, plus our own field images to close the gap between lab and field photos.
- **The AI only classifies the disease.** The pesticide comes from a lookup table. The values in the server are **placeholders** and must be checked against ICAR or agriculture department guidelines before any real use.
- Prototype trials use **water with food colour** (one colour per tank), not real pesticide.

## Known issues

| Issue | Status |
|---|---|
| Wheels: `A` moved forward, `D` backward, `W` and `S` did nothing | Open: probably reversed motor wires on one side, or a damaged board |
| Fumes from the Uno / HW-130 | Open: cause not confirmed. Over-voltage from the 3S pack is the top suspect; an overloaded L293D or a wiring short are others |
| Uno disconnects ("Input/output error") | Open: probably a power dip when servos start (not confirmed) |
| Server must save received images | Add save code to `/predict` |

## Estimates (not measured)

| Item | Estimate |
|---|---|
| Battery | 27.75 Wh; about 2.5 h continuous, 3-4 h with stops |
| Hotspot range | 10-20 m reliable; keep demos within 15 m |
| Upload time | 1-3 s at 5-15 m |
| Live view delay | about 0.3 s |
| Cost | about Rs 11,000 (team estimate) |

## Roadmap

1. Make the power safe: inspect the board, add a buck converter, give the servos a separate 5 V supply
2. Fix the wheel directions and test driving
3. Wire and test sprayer 2 and the moisture probe
4. Add the disease-to-pesticide-to-sprayer lookup, then turn `AUTO_SPRAY` on
5. Add sensing and autonomy: ultrasonic sensors, IMU, encoders and PID row following
6. Fit real pumps and tanks, calibrate the dose, and test on a mock crop bed
7. Longer term: on-device inference, GPS or visual localization, dual-rover version, cloud dashboard

## Team

| Name | Role |
|---|---|
| Pranav Sai | Hardware and embedded systems |
| Yashaswin | Hardware and embedded systems |
| Rohit | Hardware and embedded systems |
| Venkatesh | Hardware and embedded systems |
| Musthakeem | AI and software |

## Disclaimer

AgriVision is a prototype and a screening aid. It is not a diagnosis tool and does not prescribe pesticides. Confirm results with an agriculture expert before treating real crops.

## License

`TODO: choose a licence (for example MIT) or write "All rights reserved".`
