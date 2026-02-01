# Chapter 5: Cloud Integration

## 5.1 API Architecture
The device acts as a REST Client, communicating exclusively with the [OpenAI API](#openai-api). The interaction is broken down into a three-step "Chain of Thought."

### 5.1.1 Step 1: Transcribing Audio (Whisper)
*   **Endpoint:** `POST https://api.openai.com/v1/audio/transcriptions`
*   **Format:** `multipart/form-data`
*   **Challenge:** The ESP32 cannot load the entire file into RAM to base64 encode it.
*   **Solution:** We stream the binary data from [PSRAM](#psram) directly into the HTTP Client.
    *   **Boundary Construction:** We manually construct the HTTP body boundary in C++:
        ```cpp
        String head = "--boundary\r\nContent-Disposition: form-data; name=\"file\"; filename=\"audio.wav\"\r\nContent-Type: audio/wav\r\n\r\n";
        String tail = "\r\n--boundary--\r\n";
        ```
    *   **Transmission:** Send `head` -> Stream [PSRAM](#psram) Buffer -> Send `tail`.

### 5.1.2 Step 2: Intelligence (Chat Completion)
*   **Endpoint:** `POST https://api.openai.com/v1/chat/completions`
*   **Model:** `gpt-4o-mini` (Recommended for speed/cost balance) or `gpt-3.5-turbo`.
*   **Context Management:** To maintain a "conversation," the device must store the last 2-3 turns of chat history in a `std::vector<String>` and send them with each new request.
*   **System Prompt:** A hidden instruction sent with every call: *"You are a helpful, funny assistant for a 6-year-old child. Keep answers under 2 sentences."*

### 5.1.3 Step 3: Voice Synthesis (TTS)
*   **Endpoint:** `POST https://api.openai.com/v1/audio/speech`
*   **Format:** Request `mp3` or `aac` (lower bandwidth) rather than `wav`.
*   **Optimization:** Set `stream=true`. The [Net_Task](#freertos) begins filling the audio ring buffer as soon as the first 1KB of data arrives, rather than waiting for the whole file.

## 5.2 Security & Encryption
*   **TLS 1.2:** The [ESP32-S3](#esp32-s3-soc) uses the MbedTLS library.
*   **Root CA:** The application must embed the **DigiCert Global Root G2** certificate.
    *   *Risk:* If OpenAI changes their certificate authority, the device will fail to connect.
    *   *Mitigation:* Store the Root CA in [NVS](#nvs) (Non-Volatile Storage) so it can be updated via OTA without recompiling the firmware.
