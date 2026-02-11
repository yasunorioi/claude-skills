# ESP32-CAM Timelapse Builder - Skill Definition

**Skill ID**: `esp32-cam-timelapse-builder`
**Category**: Firmware / IoT / Camera
**Version**: 1.0.0
**Created**: 2026-02-06
**Invocation**: `/esp32-cam-timelapse-builder` or `/timelapse`

---

## Overview

This skill generates customized ESP32-CAM timelapse firmware for agricultural and general monitoring applications. It produces Arduino/PlatformIO compatible code with configurable capture intervals, storage options, and MQTT integration.

**Use Cases:**
- Crop growth monitoring
- Security/surveillance timelapse
- Weather observation
- Construction progress documentation
- Nature/wildlife observation

---

## Input Parameters

### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `project_name` | Camera/project identifier | house1-cam |
| `wifi_ssid` | WiFi network name | MyNetwork |
| `wifi_password` | WiFi password | MyPassword |

### Optional Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `capture_interval` | Minutes between captures | 60 |
| `daylight_start` | Start hour (24h) | 6 |
| `daylight_end` | End hour (24h) | 18 |
| `resolution` | Image resolution | UXGA (1600x1200) |
| `jpeg_quality` | JPEG quality (0-63, lower=better) | 12 |
| `storage_mode` | sd, ftp, http, or mqtt | sd |
| `mqtt_broker` | MQTT broker IP | null |
| `mqtt_port` | MQTT port | 1883 |
| `ftp_server` | FTP server address | null |
| `ftp_user` | FTP username | null |
| `ftp_password` | FTP password | null |
| `timezone_offset` | Hours from UTC | 9 (JST) |
| `ntp_server` | NTP server | ntp.nict.jp |

---

## Generated Firmware Structure

### File Structure (PlatformIO)

```
esp32-cam-timelapse/
├── platformio.ini
├── src/
│   └── main.cpp
├── include/
│   └── config.h
└── README.md
```

### File Structure (Arduino IDE)

```
esp32-cam-timelapse/
├── esp32-cam-timelapse.ino
├── config.h
└── README.md
```

---

## Configuration Template (config.h)

```cpp
#ifndef CONFIG_H
#define CONFIG_H

// ========================================
// Project Configuration
// ========================================
#define PROJECT_NAME     "{project_name}"
#define CAMERA_ID        "{camera_id}"

// ========================================
// WiFi Configuration
// ========================================
#define WIFI_SSID        "{wifi_ssid}"
#define WIFI_PASSWORD    "{wifi_password}"
#define WIFI_TIMEOUT_SEC 30

// ========================================
// Timing Configuration
// ========================================
#define CAPTURE_INTERVAL_MIN  {capture_interval}  // Minutes
#define DAYLIGHT_START_HOUR   {daylight_start}    // 24h format
#define DAYLIGHT_END_HOUR     {daylight_end}      // 24h format
#define TIMEZONE_OFFSET       {timezone_offset}   // Hours from UTC

// ========================================
// Camera Configuration
// ========================================
// Resolution options:
//   FRAMESIZE_QVGA    (320x240)
//   FRAMESIZE_VGA     (640x480)
//   FRAMESIZE_SVGA    (800x600)
//   FRAMESIZE_XGA     (1024x768)
//   FRAMESIZE_SXGA    (1280x1024)
//   FRAMESIZE_UXGA    (1600x1200)
#define CAMERA_RESOLUTION  {resolution}
#define JPEG_QUALITY       {jpeg_quality}  // 0-63 (lower = better quality)

// ========================================
// Storage Configuration
// ========================================
// Options: "sd", "ftp", "http", "mqtt"
#define STORAGE_MODE       "{storage_mode}"

// SD Card Settings
#define SD_PATH_PREFIX     "/timelapse"

// FTP Settings
#define FTP_SERVER         "{ftp_server}"
#define FTP_PORT           21
#define FTP_USER           "{ftp_user}"
#define FTP_PASSWORD       "{ftp_password}"
#define FTP_PATH           "/timelapse/{project_name}"

// HTTP Upload Settings
#define HTTP_UPLOAD_URL    "{http_url}"

// ========================================
// MQTT Configuration
// ========================================
#define MQTT_ENABLED       {mqtt_enabled}
#define MQTT_BROKER        "{mqtt_broker}"
#define MQTT_PORT          {mqtt_port}
#define MQTT_USER          ""
#define MQTT_PASSWORD      ""
#define MQTT_TOPIC_BASE    "greenhouse/{house_id}/camera"
#define MQTT_CLIENT_ID     "{project_name}"

// ========================================
// NTP Configuration
// ========================================
#define NTP_SERVER         "{ntp_server}"

// ========================================
// Deep Sleep Configuration
// ========================================
#define uS_TO_S_FACTOR     1000000ULL
#define SLEEP_DURATION_SEC (CAPTURE_INTERVAL_MIN * 60)

// ========================================
// Hardware Pins (AI-Thinker ESP32-CAM)
// ========================================
#define PWDN_GPIO_NUM      32
#define RESET_GPIO_NUM     -1
#define XCLK_GPIO_NUM       0
#define SIOD_GPIO_NUM      26
#define SIOC_GPIO_NUM      27
#define Y9_GPIO_NUM        35
#define Y8_GPIO_NUM        34
#define Y7_GPIO_NUM        39
#define Y6_GPIO_NUM        36
#define Y5_GPIO_NUM        21
#define Y4_GPIO_NUM        19
#define Y3_GPIO_NUM        18
#define Y2_GPIO_NUM         5
#define VSYNC_GPIO_NUM     25
#define HREF_GPIO_NUM      23
#define PCLK_GPIO_NUM      22

// Flash LED (optional)
#define FLASH_GPIO_NUM      4
#define FLASH_ENABLED      false

#endif // CONFIG_H
```

---

## Main Firmware Template (main.cpp)

```cpp
#include <WiFi.h>
#include <esp_camera.h>
#include <time.h>
#include "config.h"

#if MQTT_ENABLED
#include <PubSubClient.h>
WiFiClient espClient;
PubSubClient mqtt(espClient);
#endif

#if STORAGE_MODE == "sd"
#include "FS.h"
#include "SD_MMC.h"
#endif

#if STORAGE_MODE == "ftp"
#include <ESP32_FTPClient.h>
ESP32_FTPClient ftp(FTP_SERVER, FTP_USER, FTP_PASSWORD, 5000, 2);
#endif

// ========================================
// Function Prototypes
// ========================================
void initCamera();
bool connectWiFi();
void syncTime();
bool isDaylightHours();
camera_fb_t* captureImage();
bool saveToSD(camera_fb_t* fb, const char* path);
bool uploadToFTP(camera_fb_t* fb, const char* filename);
void publishMQTT(const char* filename, size_t size);
void goToDeepSleep();
void getTimestamp(char* buffer, size_t len);

// ========================================
// Setup
// ========================================
void setup() {
  Serial.begin(115200);
  Serial.println("\n========================================");
  Serial.printf("ESP32-CAM Timelapse: %s\n", PROJECT_NAME);
  Serial.println("========================================");

  // Connect to WiFi
  if (!connectWiFi()) {
    Serial.println("WiFi connection failed, sleeping...");
    goToDeepSleep();
  }

  // Sync time via NTP
  syncTime();

  // Check if daylight hours
  if (!isDaylightHours()) {
    Serial.println("Outside daylight hours, sleeping...");
    goToDeepSleep();
  }

  // Initialize camera
  initCamera();

  // Wait for camera warmup
  delay(2000);

  // Capture image
  camera_fb_t* fb = captureImage();
  if (!fb) {
    Serial.println("Capture failed, sleeping...");
    goToDeepSleep();
  }

  // Generate filename with timestamp
  char timestamp[20];
  getTimestamp(timestamp, sizeof(timestamp));

  char filename[64];
  sprintf(filename, "%s.jpg", timestamp);

  Serial.printf("Captured: %s (%d bytes)\n", filename, fb->len);

  // Save/upload based on storage mode
  bool success = false;

#if STORAGE_MODE == "sd"
  char path[80];
  char dateDir[32];

  // Get date for directory
  struct tm timeinfo;
  getLocalTime(&timeinfo);
  sprintf(dateDir, "%s/%04d%02d%02d",
          SD_PATH_PREFIX,
          timeinfo.tm_year + 1900,
          timeinfo.tm_mon + 1,
          timeinfo.tm_mday);

  sprintf(path, "%s/%s", dateDir, filename);

  // Initialize SD card
  if (SD_MMC.begin()) {
    SD_MMC.mkdir(dateDir);
    success = saveToSD(fb, path);
  }
#endif

#if STORAGE_MODE == "ftp"
  success = uploadToFTP(fb, filename);
#endif

  // Publish MQTT notification
#if MQTT_ENABLED
  if (success) {
    publishMQTT(filename, fb->len);
  }
#endif

  // Return frame buffer
  esp_camera_fb_return(fb);

  // Enter deep sleep
  goToDeepSleep();
}

void loop() {
  // Not reached due to deep sleep
}

// ========================================
// WiFi Connection
// ========================================
bool connectWiFi() {
  Serial.printf("Connecting to WiFi: %s\n", WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  int timeout = WIFI_TIMEOUT_SEC;
  while (WiFi.status() != WL_CONNECTED && timeout > 0) {
    delay(1000);
    Serial.print(".");
    timeout--;
  }
  Serial.println();

  if (WiFi.status() == WL_CONNECTED) {
    Serial.printf("Connected! IP: %s\n", WiFi.localIP().toString().c_str());
    return true;
  }
  return false;
}

// ========================================
// Time Synchronization
// ========================================
void syncTime() {
  configTime(TIMEZONE_OFFSET * 3600, 0, NTP_SERVER);
  Serial.print("Syncing time...");

  struct tm timeinfo;
  int retry = 10;
  while (!getLocalTime(&timeinfo) && retry > 0) {
    delay(500);
    retry--;
  }

  if (retry > 0) {
    Serial.printf(" %04d-%02d-%02d %02d:%02d:%02d\n",
                  timeinfo.tm_year + 1900, timeinfo.tm_mon + 1, timeinfo.tm_mday,
                  timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec);
  } else {
    Serial.println(" Failed!");
  }
}

// ========================================
// Daylight Check
// ========================================
bool isDaylightHours() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    return true; // Fail-safe: capture if time unknown
  }

  int hour = timeinfo.tm_hour;
  return (hour >= DAYLIGHT_START_HOUR && hour < DAYLIGHT_END_HOUR);
}

// ========================================
// Camera Initialization
// ========================================
void initCamera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  config.frame_size = CAMERA_RESOLUTION;
  config.jpeg_quality = JPEG_QUALITY;
  config.fb_count = 1;

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed: 0x%x\n", err);
  } else {
    Serial.println("Camera initialized");
  }
}

// ========================================
// Image Capture
// ========================================
camera_fb_t* captureImage() {
#if FLASH_ENABLED
  digitalWrite(FLASH_GPIO_NUM, HIGH);
  delay(100);
#endif

  camera_fb_t* fb = esp_camera_fb_get();

#if FLASH_ENABLED
  digitalWrite(FLASH_GPIO_NUM, LOW);
#endif

  return fb;
}

// ========================================
// SD Card Save
// ========================================
bool saveToSD(camera_fb_t* fb, const char* path) {
  File file = SD_MMC.open(path, FILE_WRITE);
  if (!file) {
    Serial.printf("Failed to open file: %s\n", path);
    return false;
  }

  size_t written = file.write(fb->buf, fb->len);
  file.close();

  if (written == fb->len) {
    Serial.printf("Saved to SD: %s\n", path);
    return true;
  }
  return false;
}

// ========================================
// FTP Upload
// ========================================
bool uploadToFTP(camera_fb_t* fb, const char* filename) {
  ftp.OpenConnection();

  // Create directory structure
  struct tm timeinfo;
  getLocalTime(&timeinfo);

  char remotePath[128];
  sprintf(remotePath, "%s/%04d%02d%02d",
          FTP_PATH,
          timeinfo.tm_year + 1900,
          timeinfo.tm_mon + 1,
          timeinfo.tm_mday);

  ftp.ChangeWorkDir(remotePath);
  ftp.InitFile("Type I");
  ftp.NewFile(filename);
  ftp.WriteData(fb->buf, fb->len);
  ftp.CloseFile();
  ftp.CloseConnection();

  Serial.printf("Uploaded to FTP: %s/%s\n", remotePath, filename);
  return true;
}

// ========================================
// MQTT Notification
// ========================================
void publishMQTT(const char* filename, size_t size) {
#if MQTT_ENABLED
  if (!mqtt.connected()) {
    mqtt.setServer(MQTT_BROKER, MQTT_PORT);
    mqtt.connect(MQTT_CLIENT_ID);
  }

  if (mqtt.connected()) {
    char topic[64];
    sprintf(topic, "%s/captured", MQTT_TOPIC_BASE);

    struct tm timeinfo;
    getLocalTime(&timeinfo);

    char payload[256];
    sprintf(payload,
            "{\"timestamp\":\"%04d-%02d-%02dT%02d:%02d:%02d\","
            "\"filename\":\"%s\","
            "\"size\":%d,"
            "\"camera_id\":\"%s\"}",
            timeinfo.tm_year + 1900, timeinfo.tm_mon + 1, timeinfo.tm_mday,
            timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec,
            filename, size, CAMERA_ID);

    mqtt.publish(topic, payload);
    Serial.printf("MQTT published: %s\n", topic);
    mqtt.disconnect();
  }
#endif
}

// ========================================
// Timestamp Generation
// ========================================
void getTimestamp(char* buffer, size_t len) {
  struct tm timeinfo;
  if (getLocalTime(&timeinfo)) {
    snprintf(buffer, len, "%02d%02d%02d",
             timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec);
  } else {
    snprintf(buffer, len, "000000");
  }
}

// ========================================
// Deep Sleep
// ========================================
void goToDeepSleep() {
  Serial.printf("Entering deep sleep for %d minutes...\n", CAPTURE_INTERVAL_MIN);
  Serial.flush();

  esp_sleep_enable_timer_wakeup(SLEEP_DURATION_SEC * uS_TO_S_FACTOR);
  esp_deep_sleep_start();
}
```

---

## PlatformIO Configuration (platformio.ini)

```ini
[env:esp32cam]
platform = espressif32
board = esp32cam
framework = arduino
monitor_speed = 115200

lib_deps =
    PubSubClient
    ESP32_FTPClient

build_flags =
    -DBOARD_HAS_PSRAM
    -mfix-esp32-psram-cache-issue
```

---

## Usage Examples

### Example 1: Basic SD Card Timelapse

```
User: /esp32-cam-timelapse-builder
      プロジェクト名: greenhouse-cam-1
      WiFi SSID: MyNetwork
      WiFi パスワード: MyPassword
      撮影間隔: 30分
      保存先: SDカード
```

**Output**: Firmware with:
- 30-minute capture interval
- SD card storage in `/timelapse/YYYYMMDD/HHMMSS.jpg`
- Daylight hours only (6:00-18:00)
- Deep sleep between captures

### Example 2: FTP Upload with MQTT

```
User: /esp32-cam-timelapse-builder
      プロジェクト名: house1-cam
      WiFi SSID: GreenNet
      WiFi パスワード: SecurePass
      撮影間隔: 60分
      保存先: FTP
      FTPサーバー: 192.168.1.100
      FTPユーザー: timelapse
      FTPパスワード: ftppass
      MQTTブローカー: 192.168.1.50
```

**Output**: Firmware with:
- 60-minute capture interval
- FTP upload to NAS
- MQTT notification on capture
- Deep sleep for battery efficiency

### Example 3: Night Vision Cam

```
User: /esp32-cam-timelapse-builder
      プロジェクト名: security-cam
      WiFi SSID: HomeNet
      WiFi パスワード: HomePass
      撮影間隔: 15分
      日中開始: 0
      日中終了: 24
      Flash LED: 有効
```

**Output**: Firmware with:
- 15-minute capture interval
- 24-hour operation
- Flash LED enabled for night shots
- SD card storage

---

## Deep Sleep Power Consumption

| State | Current | Duration |
|-------|---------|----------|
| Deep Sleep | ~10 µA | Most of time |
| WiFi Connect | ~150 mA | ~5 sec |
| Camera Capture | ~200 mA | ~3 sec |
| FTP Upload | ~150 mA | ~10 sec |

**Battery Life Estimate** (18650 2600mAh, 1-hour interval):
- ~18 seconds active per hour
- ~0.75 mAh per cycle
- **Estimated: 3,400+ hours (~140 days)**

---

## MQTT Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `greenhouse/{house}/camera/captured` | Publish | JSON (capture event) |
| `greenhouse/{house}/camera/status` | Publish | JSON (node status) |
| `greenhouse/{house}/camera/command` | Subscribe | JSON (commands) |

### Capture Event Payload

```json
{
  "timestamp": "2026-02-06T09:00:00",
  "filename": "090000.jpg",
  "size": 245760,
  "camera_id": "house1-cam"
}
```

### Status Payload

```json
{
  "node_id": "house1-cam",
  "status": "online",
  "rssi": -65,
  "free_heap": 45000,
  "sd_free_mb": 12500,
  "last_capture": "2026-02-06T09:00:00",
  "capture_count_today": 4
}
```

---

## Troubleshooting Guide

### Camera Initialization Failed

```
Error: Camera init failed: 0x20001
```

**Solutions:**
1. Check camera ribbon cable connection
2. Verify pin definitions match board variant
3. Reduce frame_size (try FRAMESIZE_SVGA)
4. Add `config.fb_count = 2` for PSRAM boards

### WiFi Connection Timeout

**Solutions:**
1. Verify SSID/password
2. Check WiFi signal strength at camera location
3. Increase `WIFI_TIMEOUT_SEC`
4. Try static IP configuration

### SD Card Mount Failed

**Solutions:**
1. Format SD card as FAT32
2. Use 16GB or smaller card
3. Check SD_MMC pin conflicts
4. Try slower SD clock speed

### Images Too Dark/Bright

**Solutions:**
1. Adjust `JPEG_QUALITY` (lower = better)
2. Enable auto-exposure: `sensor->set_exposure_ctrl(sensor, 1)`
3. Set specific exposure: `sensor->set_aec_value(sensor, 300)`
4. Adjust gain: `sensor->set_agc_gain(sensor, 10)`

---

## FFmpeg Timelapse Video Generation

### Daily Video

```bash
ffmpeg -framerate 12 -pattern_type glob \
  -i '/path/to/timelapse/20260206/*.jpg' \
  -c:v libx264 -pix_fmt yuv420p \
  /path/to/videos/20260206.mp4
```

### Weekly Compilation

```bash
ffmpeg -framerate 24 -pattern_type glob \
  -i '/path/to/timelapse/202602*/*.jpg' \
  -c:v libx264 -crf 23 -preset medium \
  /path/to/videos/week_06.mp4
```

### With Timestamp Overlay

```bash
ffmpeg -framerate 12 -pattern_type glob \
  -i '/path/to/timelapse/20260206/*.jpg' \
  -vf "drawtext=fontfile=/path/to/font.ttf:fontsize=24:fontcolor=white:x=10:y=10:text='%{pts\:localtime\:$(date +%s -d "2026-02-06 06:00:00")}'" \
  -c:v libx264 -pix_fmt yuv420p \
  /path/to/videos/20260206_timestamp.mp4
```

---

## Related Skills

- `iot-design-doc-generator`: Full IoT design documentation
- `pico-wifi-mqtt-template`: Pico W MQTT firmware
- `homeassistant-agri-starter`: HA integration patterns

---

## References

- [ESP32-CAM Datasheet](https://www.espressif.com/en/products/modules/esp32)
- [OV2640 Camera Module](https://www.uctronics.com/download/cam_module/OV2640DS.pdf)
- [ESP32 Deep Sleep](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/sleep_modes.html)
- [PubSubClient Library](https://pubsubclient.knolleary.net/)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-02-06 | Initial release |

---

**Skill Author**: Arsprout Analysis Team
**License**: MIT
