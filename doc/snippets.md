

### ✂️ Copy-Paste Ready

Here are the exact blocks you need, cleaned up and commented for your project.

#### 1. The Setup & Memory Allocation (From `Microphone.ino`)
*Why:* The StickC S3 has limited internal RAM but lots of PSRAM (SPIRAM). You must allocate your audio buffer in PSRAM or the board will crash.

```cpp
#include <M5Unified.h>

// Define buffer size: 16kHz sample rate * 5 seconds
static constexpr const size_t record_samplerate = 16000;
static constexpr const size_t max_seconds = 5;
static constexpr const size_t record_size = record_samplerate * max_seconds;

// Pointer for the audio buffer
int16_t *rec_data;
size_t current_rec_idx = 0;

void setup() {
  auto cfg = M5.config();
  
  // CRITICAL: Ensure internal mic is used
  cfg.internal_mic = true; 
  cfg.internal_spk = true;

  M5.begin(cfg);

  // STEAL THIS: Allocate memory in PSRAM (MALLOC_CAP_SPIRAM)
  // If you use standard malloc(), it will fail for large buffers.
  rec_data = (int16_t*)heap_caps_malloc(record_size * sizeof(int16_t), MALLOC_CAP_SPIRAM);
  
  if (rec_data == NULL) {
    M5.Display.print("Mem Alloc Failed!");
    while(1);
  }
  
  memset(rec_data, 0, record_size * sizeof(int16_t));
  
  M5.Display.setRotation(1); // Landscape mode
  M5.Display.setTextSize(2);
  M5.Display.print("Ready");
}
```

#### 2. The "Half-Duplex" Switching (From `Microphone.ino`)
*Why:* The M5StickC S3 cannot use the Microphone and Speaker at the exact same time without complex I2S configuration. The safest way is to turn one off before turning the other on.

```cpp
// Helper to switch to Recording Mode
void enterRecordMode() {
    if (M5.Speaker.isEnabled()) {
        M5.Speaker.end(); // STEAL THIS: Kill speaker to free I2S bus
    }
    
    // Configure Mic for Speech (Low noise)
    auto mic_cfg = M5.Mic.config();
    mic_cfg.noise_filter_level = 2; // 0-255, 2 is decent for speech
    mic_cfg.sample_rate = record_samplerate;
    M5.Mic.config(mic_cfg);
    
    M5.Mic.begin(); // Start Mic
}

// Helper to switch to Playback Mode
void enterPlayMode() {
    if (M5.Mic.isEnabled()) {
        M5.Mic.end(); // STEAL THIS: Kill mic to free I2S bus
    }
    
    M5.Speaker.begin();
    M5.Speaker.setVolume(200); // 0-255
}
```

#### 3. The PTT Loop Logic (From `Button.ino`)
*Why:* This handles the "Hold to Record" logic perfectly using `M5.update()`.

```cpp
void loop() {
  M5.update(); // STEAL THIS: Updates button states. Mandatory.

  // --- START RECORDING ---
  if (M5.BtnA.wasPressed()) {
    enterRecordMode();
    current_rec_idx = 0;
    M5.Display.fillScreen(TFT_RED);
    M5.Display.setCursor(10, 10);
    M5.Display.setTextColor(TFT_WHITE);
    M5.Display.print("REC...");
  }

  // --- RECORDING LOOP ---
  if (M5.BtnA.isPressed()) {
    // Record in small chunks (e.g., 512 samples) to keep UI responsive
    // STEAL THIS: The record function returns true if data was captured
    if (current_rec_idx < record_size) {
        // Note: M5.Mic.record takes a pointer to where to write the data
        if (M5.Mic.record(&rec_data[current_rec_idx], 512, record_samplerate)) {
            current_rec_idx += 512;
        }
    } else {
        M5.Display.fillScreen(TFT_ORANGE);
        M5.Display.print("FULL");
    }
  }

  // --- STOP & SEND ---
  if (M5.BtnA.wasReleased()) {
    M5.Display.fillScreen(TFT_BLUE);
    M5.Display.print("Thinking...");
    
    // 1. Stop Mic
    enterPlayMode(); 
    
    // 2. Play a "Ding" sound (User Feedback)
    M5.Speaker.tone(1000, 100); 
    delay(100);
    M5.Speaker.end(); // Stop tone

    // 3. TODO: Insert your OpenAI HTTP Request here
    // sendToOpenAI(rec_data, current_rec_idx);
  }
}
```

#### 4. Audio Playback (From `Microphone.ino`)
*Why:* When you get the audio response back from OpenAI (TTS), you need to play it.

```cpp
void playRawAudio(const int16_t* audio_data, size_t size) {
    enterPlayMode();
    
    // STEAL THIS: playRaw handles the I2S timing for you
    // Args: data, length, sample_rate, stereo?, repeat_count, channel
    M5.Speaker.playRaw(audio_data, size, record_samplerate, false, 1, 0);
    
    // Wait until audio finishes
    while (M5.Speaker.isPlaying()) {
        M5.delay(1);
        M5.update(); // Keep buttons responsive
    }
}
```
