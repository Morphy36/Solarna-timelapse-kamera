
```yaml
esphome:
  name: solar-timelapse
  friendly_name: Solar Timelapse Camera
  
  # DÔLEŽITÉ: Zakážeme WiFi a API, aby zariadenie nehľadalo sieť
  # a nespotrebúvalo energiu čakaním na pripojenie.
  # Týmto sa z toho stane čisto offline zariadenie.
  on_boot:
    priority: 800
    then:
      - lambda: |-
          // --- VLOŽENÝ C++ KÓD ---
          
          // 1. Konfigurácia pinov pre AI-Thinker ESP32-CAM
          camera_config_t config;
          config.ledc_channel = LEDC_CHANNEL_0;
          config.ledc_timer = LEDC_TIMER_0;
          config.pin_d0 = 5;
          config.pin_d1 = 18;
          config.pin_d2 = 19;
          config.pin_d3 = 21;
          config.pin_d4 = 36;
          config.pin_d5 = 39;
          config.pin_d6 = 34;
          config.pin_d7 = 35;
          config.pin_xclk = 0;
          config.pin_pclk = 22;
          config.pin_vsync = 25;
          config.pin_href = 23;
          config.pin_sscb_sda = 26;
          config.pin_sscb_scl = 27;
          config.pin_pwdn = 32;
          config.pin_reset = -1;
          config.xclk_freq_hz = 20000000;
          config.pixel_format = PIXFORMAT_JPEG;
          
          // Inicializácia s UXGA pre buffer
          if(psramFound()){
            config.frame_size = FRAMESIZE_UXGA;
            config.jpeg_quality = 10;
            config.fb_count = 2;
          } else {
            config.frame_size = FRAMESIZE_SVGA;
            config.jpeg_quality = 12;
            config.fb_count = 1;
          }
          
          // Inicializácia kamery
          esp_err_t err = esp_camera_init(&config);
          if (err != ESP_OK) {
            ESP_LOGE("custom", "Camera init failed 0x%x", err);
            return;
          }

          // 2. Inicializácia SD karty (SD_MMC)
          if(!SD_MMC.begin()){
            ESP_LOGE("custom", "SD Card Mount Failed");
            return;
          }
          
          // 3. Kontrola jasu (Deň vs Noc)
          sensor_t * s = esp_camera_sensor_get();
          s->set_framesize(s, FRAMESIZE_QVGA); // Malé rozlíšenie na test
          delay(100);
          
          // Zahodíme pár frameov na stabilizáciu
          for(int i=0; i<3; i++) {
            camera_fb_t * fb_temp = esp_camera_fb_get();
            esp_camera_fb_return(fb_temp);
          }

          camera_fb_t * fb = esp_camera_fb_get();
          if (!fb) {
             ESP_LOGE("custom", "Capture failed");
             return;
          }
          
          long totalBrightness = 0;
          for (int i = 0; i < fb->len; i += 50) {
            totalBrightness += fb->buf[i];
          }
          int avgBrightness = totalBrightness / (fb->len / 50);
          esp_camera_fb_return(fb);
          
          ESP_LOGI("custom", "Brightness: %d", avgBrightness);

          // HRANICA JASU (NIGHT_THRESHOLD) - Tu nastav citlivosť (napr. 15)
          if (avgBrightness > 15) {
            ESP_LOGI("custom", "It is DAY. Taking photo...");
            
            // Prepnutie na vysoké rozlíšenie
            s->set_framesize(s, FRAMESIZE_UXGA); 
            delay(500);

            fb = esp_camera_fb_get();
            if(!fb) {
              ESP_LOGE("custom", "Final capture failed");
            } else {
              // Získanie poradového čísla z globálnej premennej ESPHome
              int current_pic = id(picture_counter);
              
              // Formátovanie názvu: /foto_123.jpg
              std::string path = "/foto_" + to_string(current_pic) + ".jpg";
              
              File file = SD_MMC.open(path.c_str(), FILE_WRITE);
              if(file){
                file.write(fb->buf, fb->len);
                file.close();
                ESP_LOGI("custom", "Photo saved: %s", path.c_str());
                
                // Inkrementácia počítadla
                id(picture_counter) += 1;
                id(save_counter_to_flash).execute(); // Uloženie do flash pamäte
              } else {
                ESP_LOGE("custom", "Could not open file for writing");
              }
              esp_camera_fb_return(fb);
            }
          } else {
            ESP_LOGI("custom", "It is NIGHT. Skipping photo.");
          }

          // 4. Uspanie zariadenia
          // Časovač spánku voláme cez ESPHome deep_sleep komponent
          id(deep_sleep_1).enter_deep_sleep();

esp32:
  board: esp32dev
  framework:
    type: arduino

# Potrebné knižnice pre C++ kód
includes:
  - esp_camera.h
  - FS.h
  - SD_MMC.h
  - soc/soc.h
  - soc/rtc_cntl_reg.h

# Globálna premenná pre číslovanie fotiek (uloží sa aj po vypnutí)
globals:
  - id: picture_counter
    type: int
    restore_value: yes
    initial_value: '1'

# Skript na uloženie premennej do pamäte (aby prežila deep sleep)
script:
  - id: save_counter_to_flash
    mode: restart
    then:
      - globals.set:
          id: picture_counter
          value: !lambda 'return id(picture_counter);'

# Nastavenie Deep Sleep (na 60 minút)
deep_sleep:
  id: deep_sleep_1
  run_duration: 20s # Bezpečnostná poistka: ak sa zasekne kód, po 20s sa aj tak uspí
  sleep_duration: 60min

logger:
```
