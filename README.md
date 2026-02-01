# kidbot

### Install UART driver
https://github.com/WCHSoftGroup/ch34xser_macos

### Other drivers
https://docs.m5stack.com/en/download

### Emulator
https://wokwi.com/projects/new/esp32-s3

### UniFlow 2
https://uiflow2.m5stack.com/

### ChatBot example
https://docs.m5stack.com/en/core/AtomS3R-AI%20Chatbot
https://docs.m5stack.com/en/guide/realtime/openai/atomic_echo_base

### Simple example
https://github.com/m5stack/openai-realtime-embedded-sdk



### Phase 1: Install the Toolchain (VS Code + PlatformIO)

We will not use the Arduino IDE. We will use **PlatformIO**, which is the industry standard for embedded C++ development.

1.  **Install VS Code:**
    If you haven't already, download it from [code.visualstudio.com](https://code.visualstudio.com/).

2.  **Install PlatformIO Extension:**
    *   Open VS Code.
    *   Click the **Extensions** icon (blocks on the left) or press `Cmd+Shift+X`.
    *   Search for `PlatformIO IDE`.
    *   Click **Install**.
    *   *Wait:* It will take a few minutes to install the core Python virtual environment and toolchains. You will see a prompt to "Reload Window" when finished.

### Phase 2: Install the USB Driver

The M5StickC S3 usually uses the **CH9102** USB-to-Serial chip. macOS often needs a driver to see the COM port.

1.  **Check if needed:**
    *   Plug your M5StickC S3 into your Mac.
    *   Open Terminal and run:
        ```bash
        ls /dev/cu.*
        ```
    *   Look for something like `/dev/cu.usbserial-xxxx` or `/dev/cu.wchusbserialxxxx`.
    *   *If you see it:* You are good. Skip to Phase 3.
    *   *If you don't see it:* Proceed to step 2.

2.  **Install Driver:**
    *   Download the **CH9102 driver** for macOS.
    *   **Direct Link (M5Stack Official):** [CH9102_VCP_SER_MacOS_v1.7.zip](https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/drivers/CH9102_VCP_SER_MacOS_v1.7.zip)
    *   Unzip and run the installer.
    *   **Important:** macOS Security will likely block it. Go to **System Settings > Privacy & Security** and click "Allow" for the driver.
    *   Restart your Mac if prompted.

### Phase 3: Create Your Project

Now we create a project with "Commercial Grade" dependency locking.

1.  Open VS Code.
2.  Click the **PlatformIO Alien Icon** on the left sidebar.
3.  Click **"Create New Project"**.
4.  **Name:** `M5StickS3-AI-Chatbot`
5.  **Board:** Type `M5StickC` and select `M5Stack StickC (Espressif ESP32-PICO-D4)`.
    *   *Note:* PlatformIO doesn't have a dedicated "StickC S3" board ID yet. We select the generic StickC and then force the S3 settings in the config file. This is standard practice.
6.  **Framework:** `Arduino`.
7.  **Location:** Uncheck "Use default location" and save it in your Documents folder so you can find it easily.

### Phase 4: Configure `platformio.ini` (The Critical Step)

This is where we define the hardware and lock dependencies.

1.  In VS Code, open the file named `platformio.ini` in your project root.
2.  **Delete everything** in that file and paste this exact configuration:

```ini
; PlatformIO Project Configuration File
; Commercial Grade Setup for M5StickC S3
; ----------------------------------------------------------------

[env:m5stick-c-s3]
platform = espressif32 @ 6.5.0
board = esp32-s3-devkitc-1 ; We use a generic S3 board definition as base
framework = arduino

; --- Hardware Settings ---
; Force the CPU speed and Flash mode for S3
board_build.mcu = esp32s3
board_build.f_cpu = 240000000L
board_build.arduino.memory_type = qio_opi 
; ^ Important: S3 uses OPI PSRAM. If this is wrong, it crashes.

; --- Partition Scheme ---
; "huge_app" gives 3MB to App, 1MB to Files. Needed for SSL/Audio.
board_build.partitions = huge_app.csv

; --- Upload & Monitor ---
upload_speed = 1500000
monitor_speed = 115200
monitor_filters = esp32_exception_decoder, time ; Helps decode crash dumps

; --- Build Flags ---
build_flags = 
    -DARDUINO_USB_CDC_ON_BOOT=1  ; Essential for S3 built-in USB to show Serial output
    -DARDUINO_USB_MODE=1
    -DBOARD_HAS_PSRAM

; --- Dependencies (Locked Versions) ---
lib_deps =
    m5stack/M5Unified @ 0.1.17
    m5stack/M5GFX @ 0.1.17
    bblanchon/ArduinoJson @ 7.0.3
    earlephilhower/ESP8266Audio @ 1.9.7
```

### Phase 5: The "Hello World" Test

Let's verify the hardware, screen, and button.

1.  Open `src/main.cpp`.
2.  Replace the content with this test code:

```cpp
#include <M5Unified.h>

void setup() {
    // 1. Initialize Config
    auto cfg = M5.config();
    cfg.serial_baudrate = 115200;
    M5.begin(cfg);

    // 2. Setup Screen
    M5.Display.setRotation(1); // Landscape
    M5.Display.setTextSize(2);
    M5.Display.fillScreen(TFT_BLUE);
    M5.Display.setCursor(10, 10);
    M5.Display.print("Setup Done!");
    
    // 3. Debug Output
    Serial.println("M5StickC S3 Initialized");
}

void loop() {
    M5.update(); // Update button states

    if (M5.BtnA.wasPressed()) {
        M5.Display.fillScreen(TFT_RED);
        M5.Display.setCursor(10, 10);
        M5.Display.print("Pressed!");
        Serial.println("Button A Pressed");
    }

    if (M5.BtnA.wasReleased()) {
        M5.Display.fillScreen(TFT_BLUE);
        M5.Display.setCursor(10, 10);
        M5.Display.print("Released!");
    }
}
```

### Phase 6: Build and Upload

1.  **Connect:** Plug in your M5StickC S3.
2.  **Build:** Click the **Checkmark (✓)** icon in the bottom blue bar of VS Code.
    *   *First time:* It will download all the libraries defined in `platformio.ini`. This takes time.
3.  **Upload:** Click the **Arrow (→)** icon in the bottom blue bar.
    *   *Troubleshooting:* If upload fails, hold the **BtnG0** (the side button labeled G0 or Boot) on the Stick while plugging it in to force "Download Mode".
4.  **Monitor:** Click the **Plug (🔌)** icon in the bottom blue bar to see the Serial output.

### Summary of Commands (Terminal)

If you prefer using the terminal inside VS Code:

```bash
# 1. Initialize project (if not using UI)
# pio project init --board esp32-s3-devkitc-1

# 2. Clean previous builds (if things get weird)
pio run --target clean

# 3. Build the firmware
pio run

# 4. Upload firmware
pio run --target upload

# 5. Open Serial Monitor
pio device monitor
```


