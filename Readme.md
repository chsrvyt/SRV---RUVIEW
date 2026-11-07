# Wi-Fi RSSI-Based Patient Presence Monitoring Prototype

A low-cost ESP32 prototype that detects changes in Wi-Fi Received Signal Strength Indicator (RSSI) and uses those changes as an indication of presence in a monitored area. When the presence condition is met, an ESP32-CAM captures an image, stores it on a microSD card, and makes the saved images accessible through a small web gallery. An ESP32 DevKit V1 displays the current system state on an SSD1306 OLED.

> **Important:** This is an experimental student prototype, not a certified medical device. It does not identify a patient, detect falls, measure vital signs, or provide clinically validated alerts. Do not use it as a replacement for clinical observation or a hospital monitoring system.

## 1. What the prototype does

- Measures Wi-Fi RSSI at the ESP32 DevKit V1.
- Calibrates a baseline RSSI value while the monitored zone is empty.
- Calculates the difference between the current RSSI average and the baseline.
- Requires three consecutive qualifying measurement windows before changing to `PRESENCE`.
- Sends an HTTP request to the ESP32-CAM when presence is detected.
- Captures a JPEG image and saves it to the camera board's microSD card.
- Displays the system state and basic status information on the OLED.
- Serves a browser-based gallery for viewing and downloading stored images.

The sensing method used in this prototype is **RSSI-based**. It does not extract Wi-Fi Channel State Information (CSI).

## 2. System architecture

```text
                   Wi-Fi signal / RSSI link
  Monitored area  <---------------------------->

  ESP32-CAM (AI Thinker)       ESP32 DevKit V1
  - Camera capture             - Wi-Fi RSSI sampling
  - JPEG conversion            - Baseline comparison
  - microSD storage            - Presence state logic
  - HTTP web server  <-------> - HTTP capture request
                               - SSD1306 OLED status

  Browser  ------------------> ESP32-CAM web server
          View / download images from the gallery
```

The ESP32-CAM hosts the access point and capture/gallery endpoints. The DevKit connects to that Wi-Fi network, measures RSSI, and requests a capture when the detection condition is met.

## 3. Hardware used

| Component | Role |
|---|---|
| ESP32 DevKit V1 | RSSI measurement, detection logic, HTTP client, OLED control |
| ESP32-CAM AI-Thinker | Image capture, JPEG conversion, HTTP server |
| OV2640 camera (on ESP32-CAM) | Captures the monitored scene |
| SSD1306 OLED, 128 × 64 | Displays presence and system status |
| microSD card (1 GB used in the prototype) | Stores captured JPEG images |
| USB power / suitable 5 V supply | Powers the boards |

## 4. Connections

### OLED to ESP32 DevKit V1

| OLED pin | ESP32 DevKit V1 |
|---|---|
| VCC | 3.3 V |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

The OLED uses I2C, commonly at address `0x3C` in this setup.

### ESP32-CAM and microSD

The microSD card is inserted into the ESP32-CAM's onboard card socket. The camera firmware mounts it using `SD_MMC` in 1-bit mode:

```cpp
SD_MMC.begin("/sdcard", true);
```

The camera board's camera and SD interfaces use the board's assigned pins; do not wire the card socket as an external SPI module.

## 5. Wi-Fi configuration

The prototype configuration used during development was:

| Setting | Value |
|---|---|
| Access-point SSID | `CSI_ZONE` |
| Access-point IP | `192.168.4.1` |
| HTTP port | `80` |

The password used in the development sketch was `ZoneTest8266`. **Change this password before using the project outside a private test setup.** Keep credentials in a local configuration file rather than publishing them in a public repository.

The DevKit connects to the ESP32-CAM access point in station mode. The RSSI returned by `WiFi.RSSI()` is used by the detector.

## 6. Detection logic

The prototype uses these development settings:

| Parameter | Value |
|---|---:|
| Baseline calibration period | 10 seconds |
| RSSI samples per measurement window | 20 |
| Delay between RSSI samples | 50 ms |
| RSSI deviation threshold | 5 dB |
| Consecutive qualifying windows | 3 |
| Clear delay | 3 seconds |

For a measurement window:

\[
RSSI_{avg} = \frac{1}{N}\sum_{i=1}^{N} RSSI_i
\]

The deviation is calculated relative to the calibrated baseline:

\[
\Delta = |RSSI_{avg} - RSSI_{baseline}|
\]

The state changes from `CLEAR` to `PRESENCE` after three consecutive windows meet or exceed the 5 dB threshold. After the deviation falls below the threshold, the system waits for the configured 3-second clear interval before returning to `CLEAR`.

These are prototype settings, not clinically optimized thresholds. RSSI is affected by distance, walls, multipath propagation, antenna orientation, interference, and movement of objects in the environment.

## 7. Image capture and web endpoints

The ESP32-CAM firmware provides these endpoints:

| Endpoint | Purpose |
|---|---|
| `/` | Basic camera server page / root route |
| `/capture` | Capture an image, save it to microSD, and return the JPEG response |
| `/gallery` | Browser-based image gallery |
| `/image?name=IMG_0001.jpg` | View a stored image |
| `/download?name=IMG_0001.jpg` | Download a stored image |

The camera firmware uses sequential filenames such as `IMG_0001.jpg`. The `/capture` handler returns an HTTP response and includes custom status headers such as `X-SD-Saved` and `X-Image-Name`.

The current DevKit logic checks the HTTP response status. It does **not** parse those custom headers, so the OLED should not be interpreted as an independently verified SD-card write indicator unless that header parsing is added.

To open the gallery, connect a phone or computer to the ESP32-CAM access point and visit:

```text
http://192.168.4.1/gallery
```

## 8. Software and libraries

Developed using Arduino IDE and the ESP32 Arduino core.

Libraries used by the sketches include:

- `WiFi.h` — Wi-Fi connectivity and RSSI
- `WebServer.h` — ESP32-CAM HTTP routes
- `HTTPClient.h` — HTTP request from the DevKit
- `esp_camera.h` — camera capture
- `SD_MMC.h` — microSD access
- `Wire.h` — I2C
- `Adafruit_GFX`
- `Adafruit_SSD1306`

Install the ESP32 board package and the OLED libraries through the Arduino IDE Library Manager. Select the appropriate board profile for the ESP32 DevKit V1 and the AI Thinker ESP32-CAM.

## 9. Build and run

1. Insert a supported, formatted microSD card into the ESP32-CAM.
2. Upload the camera/web-server sketch to the AI Thinker ESP32-CAM.
3. Open the camera serial monitor and confirm that the access point starts and the server reports its IP address.
4. Upload the RSSI/OLED detector sketch to the ESP32 DevKit V1.
5. Confirm that the OLED initializes and the DevKit connects to `CSI_ZONE`.
6. Keep the monitored zone empty during the baseline calibration period.
7. Move through the monitored zone and observe whether the RSSI deviation meets the configured trigger condition.
8. When presence is detected, confirm the HTTP capture response and check the ESP32-CAM microSD card or `/gallery` page for the saved image.

If a board fails to upload, check the selected COM port, board profile, USB cable, power supply, and BOOT/EN procedure required by that ESP32 board.

## 10. Sample data

The separate file [`sample_data.csv`](sample_data.csv) contains a small **synthetic example** of RSSI windows, baseline deviation, state transitions, and capture events.

It is provided to demonstrate the dataset format and how the threshold logic can be represented. It is **not measured data**, and must not be used to claim real-world accuracy, sensitivity, specificity, or latency.

For a real experiment, log timestamped RSSI samples from the hardware alongside ground-truth observations, capture outcomes, and environmental conditions. Keep training/demo data separate from measured evaluation data.

## 11. Current limitations

- RSSI is a coarse signal-strength measurement and is not equivalent to CSI.
- A 5 dB threshold may not generalize across rooms, distances, hardware units, or Wi-Fi environments.
- Presence detection is not the same as patient identification or activity recognition.
- Camera images may contain sensitive personal information.
- The prototype has no cloud backend, user authentication, encrypted image storage, or role-based access control.
- The current logic does not provide fall detection, vital-sign monitoring, or clinical alarm functionality.
- Performance has not been established through a controlled clinical or large-scale field evaluation.

## 12. Privacy and responsible testing

Use the camera only in a controlled environment with informed consent from people who may be captured. Avoid recording identifiable people unless it is necessary for the test and explicit permission has been obtained. Restrict access to the access point and gallery, secure the device physically, and delete test images when they are no longer needed. Obtain institutional approval before any testing in a healthcare facility.

## 13. Suggested next steps

- Record multiple trials at controlled distances and different movement conditions.
- Compare every detection event against independently recorded ground truth.
- Report precision, recall, F1-score, false-positive rate, and detection latency only after collecting sufficient real observations.
- Test threshold stability across rooms, orientations, and Wi-Fi interference conditions.
- Add authentication, access control, and secure image handling before any deployment beyond a closed demonstration.

## 14. Project status

This repository documentation describes the implemented prototype and its development configuration. Replace the example data with actual hardware logs before publishing experimental findings. Add board-specific source files, wiring photographs, and verified test results to the repository as they become available.
