# Chapter 6: Power Management

## 6.1 The Energy Budget
The [M5StickC S3](#m5stickc-s3) contains a **250mAh LiPo battery**.
*   **Active Current (Wi-Fi + Screen + CPU):** ~180mA (Runtime: ~1.2 hours).
*   **Deep Sleep Current:** ~10uA (Runtime: Months).

## 6.2 Power States
We utilize the [AXP2101](#axp2101) PMU to manage power rails dynamically.

| State | CPU Freq | Wi-Fi | Screen | PMU Action | Trigger |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Active** | 240 MHz | ON | 100% | All Rails ON | User Interaction |
| **Standby** | 80 MHz | OFF | 20% | Audio Amp OFF | 10s Inactivity |
| **Deep Sleep** | OFF | OFF | OFF | LCD/Amp OFF | 60s Inactivity |

## 6.3 Implementation Details
1.  **Dynamic Frequency Scaling:** When waiting for the user to press a button, drop CPU clock to 80MHz.
    ```cpp
    setCpuFrequencyMhz(80);
    ```
2.  **Peripheral Shutdown:** Explicitly disable the [AW8737](#aw8737) amplifier via GPIO when not playing audio to prevent "hissing" and save ~15mA.
3.  **Wake Sources:** Configure the [ESP32-S3](#esp32-s3-soc) RTC to wake up *only* on:
    *   Button A (Main Button) press.
    *   Low Battery Alarm (from [AXP2101](#axp2101)).

---

# Chapter 7: Connectivity & Provisioning

## 7.1 The "First Boot" Problem
The device cannot ship with hardcoded Wi-Fi credentials. We implement a **Captive Portal** strategy.

## 7.2 WiFiManager Implementation
1.  **Check:** On boot, attempt to connect to saved credentials in [NVS](#nvs).
2.  **Fallback:** If connection fails after 10 seconds, enter **AP Mode**.
    *   Device broadcasts SSID: `ChatAI-Setup`.
    *   Screen displays: "Connect to ChatAI-Setup on your phone."
3.  **Configuration:** User opens browser to `192.168.4.1`, scans for home Wi-Fi, enters password, and saves.
4.  **Reboot:** Device restarts and connects.

## 7.3 Reconnection Logic
Wi-Fi is unstable. The [Net_Task](#freertos) implements an **Exponential Backoff** algorithm.
*   If connection drops: Retry in 1s, then 2s, then 4s, then 8s.
*   If retry > 60s: Show "Network Error" icon and enter Standby to save battery.

---

# Chapter 8: Security & Secrets

## 8.1 API Key Protection
The OpenAI API Key is a high-value target.
*   **Development:** Use a `secrets.h` file listed in `.gitignore`.
*   **Production:** Do not embed the key in the firmware binary.
    *   *Injection:* Use a manufacturing tool to write the API Key directly into a protected [NVS](#nvs) partition during the factory flashing process.
    *   *Retrieval:* The firmware reads `preferences.getString("apikey")` at runtime.

## 8.2 Firmware Readout Protection
To prevent cloning or key extraction:
1.  **Flash Encryption:** Enable ESP32 Flash Encryption (Release Mode) in the bootloader. This encrypts the code and NVS partition using a hardware-fused key.
2.  **Secure Boot V2:** Ensures only signed firmware can run on the device, preventing malicious code injection.

---

# Chapter 9: User Experience (UX)

## 9.1 Visual Language
Children may not read text. We rely on color and animation via the [ST7789](#st7789) screen.

*   **Listening (Red):** High-contrast microphone icon. The icon scales size based on audio amplitude (Visual Feedback that "it hears me").
*   **Thinking (Blue):** A "breathing" circle animation. *Psychological Note:* This masks the 2-4 second latency of the API.
*   **Speaking (Green):** A waveform visualization that syncs with the audio output.

## 9.2 Haptic & Audio Cues
*   **Start Record:** Short high-pitch beep + Short Vibration.
*   **Stop Record:** Short low-pitch beep.
*   **Error:** "Bonk" sound + Long Vibration.

## 9.3 Latency Masking
To make the device feel faster:
1.  **Pre-connection:** Start the SSL handshake with OpenAI as soon as the button is *pressed*, not when released.
2.  **Streaming TTS:** Begin playing audio as soon as the first packet arrives, rather than waiting for the full sentence.

---

# Chapter 10: Maintenance & Updates

## 10.1 Partition Scheme
The 8MB Flash of the [ESP32-S3](#esp32-s3-soc) is divided to support updates:
*   **Factory App:** 2MB (The fallback version).
*   **OTA_0:** 2MB (The active version).
*   **OTA_1:** 2MB (The update download slot).
*   **SPIFFS/LittleFS:** 1.5MB (For storing icons and sound effects).

## 10.2 Over-the-Air (OTA) Workflow
Since the device has no user interface for typing URLs, updates are automatic.
1.  **Check:** On boot (or every 24h), query a JSON manifest hosted on GitHub/S3: `https://updates.example.com/firmware.json`.
2.  **Compare:** If `remote_version > current_version`:
3.  **Download:** Stream the `.bin` file to the passive OTA partition.
4.  **Verify:** Check MD5 checksum.
5.  **Switch:** Update boot partition and restart.

## 10.3 Rollback Mechanism
If the new firmware crashes (Watchdog Reset) 3 times within 30 seconds of booting:
*   The ESP32 bootloader automatically reverts to the previous working partition.
*   This prevents "bricking" the device in the field.
