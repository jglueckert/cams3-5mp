# M5Stack CAMS3 5MP → Frigate NVR
## Reproducible Setup Guide
*Based on confirmed working steps for the PY260/OV5640 variant*

---

## Overview

This guide covers flashing the M5Stack CAMS3 5MP (ESP32-S3 + PY260/OV5640 sensor) with custom firmware that streams MJPEG over HTTP, and integrating that stream into a locally hosted Frigate NVR instance.

> **This guide is specific to the 5MP variant.** The standard CAMS3 (1MP, GC0308 sensor) requires a different library and approach. If you are unsure which unit you have, the 5MP model is labelled "Unit CamS3-5MP" on the packaging.

---

## Requirements

### Hardware
- M5Stack CAMS3 5MP unit (ESP32-S3 + PY260/OV5640 sensor)
- USB-C data cable (charge-only cables will not work for flashing)
- WiFi router — 2.4 GHz recommended for range
- Home server running Frigate via Docker

### Software
- Arduino IDE 2.x — https://arduino.cc/en/software
- ESP32 board package by Espressif (installed via Arduino Boards Manager)
- M5Unified library (installed via Arduino Library Manager)
- Frigate NVR — https://frigate.video

---

## Step 1 — Arduino IDE Setup

### 1.1 Install Arduino IDE
Download and install Arduino IDE 2.x from https://arduino.cc/en/software.

### 1.2 Add the ESP32 Board Package
Go to **Tools → Board → Boards Manager**, search for `esp32`, and install **esp32 by Espressif Systems**.

### 1.3 Select the Correct Board
Go to **Tools → Board → esp32** and select **M5ESP32S3**.

### 1.4 Configure Board Settings
Set all of the following under the **Tools** menu:

| Setting | Value |
|---------|-------|
| PSRAM | OPI PSRAM |
| Partition Scheme | Huge APP (3MB No OTA/1MB SPIFFS) |
| USB CDC On Boot | Enabled |
| Upload Speed | 921600 |

### 1.5 Install M5Unified Library
Go to **Sketch → Include Library → Manage Libraries**, search for `M5Unified`, and install the result by M5Stack.

> The `esp32-camera` driver is bundled with the ESP32 board package — no separate install is needed.

---

## Step 2 — Detect Hardware Version

The CAMS3 5MP shipped in two hardware revisions with different internal configurations. You must identify which version you have before flashing the stream firmware.

### 2.1 Flash the Version Detector

Create a new sketch in Arduino IDE, paste the following code, and upload it:

```cpp
#include <Wire.h>

#define SIOD_GPIO_NUM 17
#define SIOC_GPIO_NUM 41

void setup() {
  Serial.begin(115200);
  delay(500);
  Wire.begin(SIOD_GPIO_NUM, SIOC_GPIO_NUM);
  Serial.println("CAMS3 5MP Hardware Version Detector");
}

void loop() {
  byte version = readRegister(0x1f, 0x0200);
  if (version == 0x01) {
    Serial.println(">>> New Version (0x01)");
  } else if (version == 0xFF) {
    Serial.println(">>> Old Version (0xFF) — patched library required");
  } else {
    Serial.print(">>> Unknown: 0x");
    Serial.println(version, HEX);
  }
  delay(1000);
}

byte readRegister(byte slaveAddr, unsigned int regAddr) {
  Wire.beginTransmission(slaveAddr);
  Wire.write((byte)(regAddr >> 8));
  Wire.write((byte)(regAddr & 0xFF));
  Wire.endTransmission(false);
  Wire.requestFrom(slaveAddr, (byte)1);
  if (Wire.available()) return Wire.read();
  return 0x00;
}
```

Open **Tools → Serial Monitor** at **115200 baud**. The output will be one of:

- `New Version (0x01)` → proceed directly to Step 3, set `HW_VERSION 1`
- `Old Version (0xFF)` → you must complete Step 2.2 before continuing

### 2.2 Old Hardware Only — Install Patched Camera Library

If your unit printed `Old Version (0xFF)`, the standard esp32-camera library cannot initialise the sensor. Follow these steps:

1. In Boards Manager, install esp32 by Espressif version **3.1.0** specifically (not the latest)
2. Download the patched library from M5Stack:
   `https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/1050/libespressif__esp32-camera.zip`
3. Extract the zip to obtain `libespressif__esp32-camera.a`
4. Replace the file at this path (substitute your Windows username):
   ```
   C:\Users\<username>\AppData\Local\Arduino15\packages\esp32\tools\
   esp32-arduino-libs\idf-release_v5.3-083aad99-v2\esp32s3\lib\
   libespressif__esp32-camera.a
   ```
5. Restart Arduino IDE

---

## Step 3 — Flash the Stream Firmware

### 3.1 Create the Sketch

In Arduino IDE, create a new sketch named `cams3_frigate_stream` and replace all contents with the following. Edit the values in the USER CONFIGURATION section before uploading.

```cpp
/**
 * M5Stack CAMS3 5MP — MJPEG Stream for Frigate NVR
 * Sensor: PY260 / OV5640
 * Stream URL: http://<CAMERA_IP>:81/stream
 * Snapshot URL: http://<CAMERA_IP>:81/capture
 */

#include "esp_camera.h"
#include "esp_http_server.h"
#include <WiFi.h>
// M5Unified intentionally omitted — the CAMS3 unit has no display,
// and M5Unified's board auto-detection repeatedly tries to claim the
// I2C bus that esp_camera already owns, causing floods of log errors.

// -----------------------------------------
//  USER CONFIGURATION — edit these values
// -----------------------------------------
#define HW_VERSION 1              // 1 = new hardware, 0 = old hardware

const char* WIFI_SSID     = "YOUR_WIFI_SSID";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";

// Resolution: FRAMESIZE_VGA (640x480) or FRAMESIZE_HD (1280x720)
// HD is recommended for Frigate object detection accuracy.
framesize_t STREAM_RESOLUTION = FRAMESIZE_HD;

int  JPEG_QUALITY  = 12;   // 10 (best) to 63 (worst)
int  STREAM_PORT   = 81;

// Optional static IP — set USE_STATIC_IP to true and fill in your values
bool USE_STATIC_IP = false;
IPAddress STATIC_IP (192, 168, 1, 200);
IPAddress GATEWAY   (192, 168, 1,   1);
IPAddress SUBNET    (255, 255, 255, 0);
IPAddress DNS1      (  8,   8,   8, 8);
// -----------------------------------------

// Pin map — confirmed working for new hardware (0x01)
// Both hardware versions use the same GPIO layout; old hardware
// requires the patched camera library (see Step 2.2).
#define PWDN_GPIO_NUM    -1
#define RESET_GPIO_NUM   21
#define XCLK_GPIO_NUM    11
#define SIOD_GPIO_NUM    17
#define SIOC_GPIO_NUM    41
#define Y9_GPIO_NUM      13
#define Y8_GPIO_NUM       4
#define Y7_GPIO_NUM      10
#define Y6_GPIO_NUM       5
#define Y5_GPIO_NUM       7
#define Y4_GPIO_NUM      16
#define Y3_GPIO_NUM      15
#define Y2_GPIO_NUM       6
#define VSYNC_GPIO_NUM   42
#define HREF_GPIO_NUM    18
#define PCLK_GPIO_NUM    12

#define PART_BOUNDARY "ov5640frame"
static const char* STREAM_CONTENT_TYPE =
  "multipart/x-mixed-replace;boundary=" PART_BOUNDARY;
static const char* STREAM_BOUNDARY = "\r\n--" PART_BOUNDARY "\r\n";
static const char* STREAM_PART =
  "Content-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n";

httpd_handle_t stream_httpd = NULL;

static esp_err_t stream_handler(httpd_req_t* req) {
  camera_fb_t* fb  = NULL;
  esp_err_t    res = ESP_OK;
  char         part_buf[128];

  res = httpd_resp_set_type(req, STREAM_CONTENT_TYPE);
  if (res != ESP_OK) return res;
  httpd_resp_set_hdr(req, "Access-Control-Allow-Origin", "*");
  httpd_resp_set_hdr(req, "Cache-Control", "no-store, no-cache");

  while (true) {
    fb = esp_camera_fb_get();
    if (!fb) {
      Serial.println("[WARN] Frame capture failed -- retrying");
      delay(100);
      continue;
    }
    res = httpd_resp_send_chunk(req, STREAM_BOUNDARY, strlen(STREAM_BOUNDARY));
    if (res != ESP_OK) break;
    size_t hlen = snprintf(part_buf, sizeof(part_buf), STREAM_PART, fb->len);
    res = httpd_resp_send_chunk(req, part_buf, hlen);
    if (res != ESP_OK) break;
    res = httpd_resp_send_chunk(req, (const char*)fb->buf, fb->len);
    esp_camera_fb_return(fb);
    fb = NULL;
    if (res != ESP_OK) break;
  }
  if (fb) esp_camera_fb_return(fb);
  return res;
}

static esp_err_t capture_handler(httpd_req_t* req) {
  camera_fb_t* fb = esp_camera_fb_get();
  if (!fb) { httpd_resp_send_500(req); return ESP_FAIL; }
  httpd_resp_set_type(req, "image/jpeg");
  httpd_resp_set_hdr(req, "Content-Disposition", "inline; filename=snap.jpg");
  httpd_resp_set_hdr(req, "Access-Control-Allow-Origin", "*");
  esp_err_t res = httpd_resp_send(req, (const char*)fb->buf, fb->len);
  esp_camera_fb_return(fb);
  return res;
}

void startServer() {
  if (stream_httpd) { httpd_stop(stream_httpd); stream_httpd = NULL; delay(100); }
  httpd_config_t config   = HTTPD_DEFAULT_CONFIG();
  config.server_port      = STREAM_PORT;
  config.max_uri_handlers = 4;
  config.lru_purge_enable = true;
  httpd_uri_t su = { "/stream",  HTTP_GET, stream_handler,  NULL };
  httpd_uri_t cu = { "/capture", HTTP_GET, capture_handler, NULL };
  if (httpd_start(&stream_httpd, &config) == ESP_OK) {
    httpd_register_uri_handler(stream_httpd, &su);
    httpd_register_uri_handler(stream_httpd, &cu);
    Serial.printf("[OK] Stream:   http://%s:%d/stream\n",
                  WiFi.localIP().toString().c_str(), STREAM_PORT);
    Serial.printf("[OK] Snapshot: http://%s:%d/capture\n",
                  WiFi.localIP().toString().c_str(), STREAM_PORT);
  }
}

bool initCamera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0; config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM; config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM; config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM; config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM; config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk     = XCLK_GPIO_NUM;  config.pin_pclk  = PCLK_GPIO_NUM;
  config.pin_vsync    = VSYNC_GPIO_NUM; config.pin_href  = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn     = PWDN_GPIO_NUM;  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  config.frame_size   = STREAM_RESOLUTION;
  config.jpeg_quality = JPEG_QUALITY;
  config.fb_count     = 2;
  config.fb_location  = CAMERA_FB_IN_PSRAM;
  config.grab_mode    = CAMERA_GRAB_LATEST;

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("[ERR] Camera init failed: 0x%x\n", err);
    return false;
  }

  sensor_t* s = esp_camera_sensor_get();
  Serial.printf("[OK] Sensor PID: 0x%04X\n", s->id.PID);

  s->set_quality(s, JPEG_QUALITY);
  s->set_framesize(s, STREAM_RESOLUTION);
  s->set_brightness(s, 0);  s->set_contrast(s, 0);
  s->set_saturation(s, 0);  s->set_sharpness(s, 0);
  s->set_denoise(s, 1);     s->set_whitebal(s, 1);
  s->set_awb_gain(s, 1);    s->set_wb_mode(s, 0);
  s->set_exposure_ctrl(s, 1); s->set_aec2(s, 1);
  s->set_ae_level(s, 0);    s->set_gain_ctrl(s, 1);
  s->set_agc_gain(s, 0);    s->set_gainceiling(s, (gainceiling_t)6);
  s->set_bpc(s, 1);         s->set_wpc(s, 1);
  s->set_raw_gma(s, 1);     s->set_lenc(s, 1);
  s->set_hmirror(s, 0);     // set to 1 if image is horizontally mirrored
  s->set_vflip(s, 0);       // set to 1 if image is upside-down
  s->set_dcw(s, 1);         s->set_colorbar(s, 0);

  Serial.println("[..] Sensor warm-up (800ms)...");
  delay(800);

  // Flush stale frames accumulated during warm-up
  for (int i = 0; i < 4; i++) {
    camera_fb_t* f = esp_camera_fb_get();
    if (f) { esp_camera_fb_return(f); } else { delay(200); }
  }

  Serial.println("[OK] Camera ready");
  return true;
}

void connectWiFi() {
  if (USE_STATIC_IP) WiFi.config(STATIC_IP, GATEWAY, SUBNET, DNS1);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  WiFi.setSleep(false);
  Serial.print("[..] Connecting to WiFi");
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 40) {
    delay(500); Serial.print("."); attempts++;
  }
  if (WiFi.status() == WL_CONNECTED) {
    Serial.printf("\n[OK] IP: %s  RSSI: %d dBm\n",
                  WiFi.localIP().toString().c_str(), WiFi.RSSI());
  } else {
    Serial.println("\n[ERR] WiFi failed -- restarting");
    delay(3000); ESP.restart();
  }
}

void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("\n=== CAMS3 5MP Frigate Stream ===");

  if (!initCamera()) {
    Serial.println("[ERR] Camera init failed -- halting");
    while (true) delay(500);
  }

  connectWiFi();
  startServer();
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("[WARN] WiFi lost -- reconnecting...");
    connectWiFi();
    startServer();
  }
  delay(5000);
}
```

### 3.2 Edit Configuration Values

Before uploading, update the following at the top of the sketch:

| Constant | What to set |
|----------|-------------|
| `HW_VERSION` | `1` for new hardware, `0` for old hardware |
| `WIFI_SSID` | Your WiFi network name |
| `WIFI_PASSWORD` | Your WiFi password |
| `STREAM_RESOLUTION` | `FRAMESIZE_HD` (1280×720) recommended |
| `USE_STATIC_IP` | Set `true` and fill in `STATIC_IP` for a fixed address |

### 3.3 Upload the Sketch

1. Connect the CAMS3 to your PC via USB-C
2. Select the correct port under **Tools → Port**
3. Click **Upload (→)**
4. If upload fails with a port error, hold the **BOOT** button on the unit, click Upload, then release BOOT once `Connecting....` appears in the console

### 3.4 Verify the Stream

Open **Tools → Serial Monitor** at **115200 baud**. You should see:

```
=== CAMS3 5MP Frigate Stream ===
[OK] Sensor PID: 0x039E
[OK] Camera ready
[..] Connecting to WiFi......
[OK] IP: 192.168.x.x  RSSI: -52 dBm
[OK] Stream:   http://192.168.x.x:81/stream
[OK] Snapshot: http://192.168.x.x:81/capture
```

Note the IP address — you will need it for the Frigate configuration.

Open a browser and navigate to `http://<CAMERA_IP>:81/stream` to confirm a live image is visible before proceeding.

> **If the image is upside-down or mirrored**, change `set_vflip` or `set_hmirror` from `0` to `1` in the `initCamera()` function and re-upload.

> **If you see `[WARN] Frame capture failed` repeatedly**, the sensor did not warm up in time. Increase the `delay(800)` value to `1200` and re-upload.

---

## Step 4 — Frigate Configuration

### 4.1 Add the Camera

Add the following to your Frigate `config.yml` under the `cameras:` block. Replace `<CAMERA_NAME>` with a meaningful label (e.g. `frontdoor`, `garage`) and `<CAMERA_IP>` with the IP from the Serial Monitor.

```yaml
cameras:
  <CAMERA_NAME>:
    ffmpeg:
      hwaccel_args: []            # required — disables VAAPI which does not
      input_args: preset-http-mjpeg-generic   # support the MJPEG profile this camera outputs
      inputs:
        - path: http://<CAMERA_IP>:81/stream
          roles:
            - detect
            - record
    detect:
      width: 1280
      height: 720
      fps: 10
    record:
      enabled: true
```

> **`hwaccel_args: []` is required.** Without it, Frigate will attempt to use VAAPI hardware acceleration to decode the MJPEG stream. The OV5640 outputs MJPEG profile 192 which VAAPI does not support, causing ffmpeg to crash immediately with `No support for codec mjpeg profile 192`.

> **`input_args: preset-http-mjpeg-generic` is required.** Without it, ffmpeg probes the stream as RTSP and fails. This preset tells ffmpeg the source is a pull-based HTTP MJPEG stream.

### 4.2 Adding Multiple Cameras

Each camera is a separate named entry under `cameras:`. Repeat Step 3 for each physical unit (each will get its own IP), then add an entry for each one:

```yaml
cameras:
  frontdoor:
    ffmpeg:
      hwaccel_args: []
      input_args: preset-http-mjpeg-generic
      inputs:
        - path: http://192.168.68.74:81/stream
          roles:
            - detect
            - record
    detect:
      width: 1280
      height: 720
      fps: 10
    record:
      enabled: true

  garage:
    ffmpeg:
      hwaccel_args: []
      input_args: preset-http-mjpeg-generic
      inputs:
        - path: http://192.168.68.55:81/stream
          roles:
            - detect
            - record
    detect:
      width: 1280
      height: 720
      fps: 10
    record:
      enabled: true
```

### 4.3 Restart Frigate

```bash
docker compose restart frigate
```

Open the Frigate UI at `http://<SERVER_IP>:5000` — the camera should appear and show a live feed within a few seconds.

---

## Step 5 — MQTT & Home Assistant Integration

### 5.1 Configure MQTT in Frigate

Add the following to your `config.yml` (not nested under `cameras`):

```yaml
mqtt:
  enabled: true
  host: <MQTT_BROKER_IP>
  port: 1883
  user: frigate          # omit if broker allows anonymous connections
  password: yourpassword # omit if broker allows anonymous connections
```

To test whether your broker requires credentials, run this from any machine on the network:

```bash
mosquitto_pub -h <MQTT_BROKER_IP> -p 1883 -t test/topic -m "hello"
```

If it succeeds without credentials, omit the `user` and `password` lines entirely.

### 5.2 Add Frigate Integration in Home Assistant

1. In Home Assistant go to **Settings → Devices & Services → Add Integration**
2. Search for **Frigate** and select it
3. Enter your Frigate server URL: `http://<SERVER_IP>:5000`
4. Each camera will appear as a device with entities for stream, snapshots, and detection events

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Upload fails — port error | USB cable is charge-only, or wrong port | Use a data USB-C cable. Hold BOOT button during upload. Check Device Manager for correct COM port. |
| `Camera init failed: 0x106` | Wrong pin map or sensor not detected | Verify PSRAM is set to OPI. Check ribbon cable is fully seated. |
| `Camera init failed: 0x103` | I2C bus conflict | Camera must init before any other library. Ensure M5Unified is not included. |
| Frames null after init | OV5640 warm-up incomplete | Increase warm-up delay from 800ms to 1200ms in `initCamera()`. |
| Frigate: `Connection timed out` | Frigate container cannot reach camera IP | Add `network_mode: host` to Frigate's docker-compose service. |
| Frigate: `mjpeg profile 192` error | VAAPI trying to decode MJPEG | Add `hwaccel_args: []` to the camera block in config.yml. |
| Frigate: `Error opening input` | Missing MJPEG input preset | Add `input_args: preset-http-mjpeg-generic` to the camera block. |
| Stream works in browser but not Frigate | ffmpeg probing wrong protocol | Confirm both `hwaccel_args: []` and `input_args` are set. |
| Image upside-down or mirrored | Camera mounted inverted | Set `set_vflip(s, 1)` and/or `set_hmirror(s, 1)` in sketch and re-flash. |
| WiFi drops regularly | Power save mode or weak signal | `WiFi.setSleep(false)` is already set. Use static IP. Move closer to router. |

---

*References: [Frigate NVR docs](https://docs.frigate.video) | [M5Stack CAMS3 5MP docs](https://docs.m5stack.com/en/arduino/m5unitcams3_5mp/program) | [ESP32-Camera driver](https://github.com/espressif/esp32-camera)*
