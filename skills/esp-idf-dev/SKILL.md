---
name: esp-idf-dev
description: Develops ESP32 firmware using Espressif's ESP-IDF framework (v5.4/5.5). Covers project scaffolding, FreeRTOS task architecture, peripheral drivers (I2C, MCPWM, ADC, GPIO, LEDC, UART, SPI), WiFi/MQTT networking, NVS storage, OTA updates, and the CMake build system. Use when writing ESP-IDF C/C++ code, configuring menuconfig/sdkconfig, creating CMakeLists.txt files, debugging ESP32 firmware, working with FreeRTOS tasks/queues/semaphores, setting up I2C sensors, controlling servos/motors with MCPWM, or connecting ESP32 to WiFi/MQTT. Also use when user mentions idf.py, esp_err_t, ESP_LOG, xTaskCreate, or any ESP-IDF API. Do NOT use for Arduino framework code or ESPHome YAML configs.
---

# ESP-IDF Development (v5.4/5.5)

## Critical: Use New Driver APIs Only

ESP-IDF v5.x introduced redesigned peripheral drivers. **Never use legacy APIs.** The legacy drivers (`driver/i2c.h`, `driver/mcpwm.h`, etc.) are removed in v6.0. Always use the new component-based drivers:

| Peripheral | New Header | Component Dependency |
|-----------|-----------|---------------------|
| I2C Master | `driver/i2c_master.h` | `esp_driver_i2c` |
| MCPWM | `driver/mcpwm_prelude.h` | `esp_driver_mcpwm` |
| GPIO | `driver/gpio.h` | `esp_driver_gpio` |
| ADC Oneshot | `esp_adc/adc_oneshot.h` | `esp_adc` |
| ADC Continuous | `esp_adc/adc_continuous.h` | `esp_adc` |
| LEDC (PWM) | `driver/ledc.h` | `esp_driver_ledc` |
| UART | `driver/uart.h` | `esp_driver_uart` |
| SPI Master | `driver/spi_master.h` | `esp_driver_spi` |
| Timer (GPTimer) | `driver/gptimer.h` | `esp_driver_gptimer` |
| RMT (IR/WS2812) | `driver/rmt_tx.h` / `driver/rmt_rx.h` | `esp_driver_rmt` |
| PCNT (Encoder) | `driver/pulse_cnt.h` | `esp_driver_pcnt` |
| Temp Sensor | `driver/temperature_sensor.h` | `esp_driver_tsens` |
| TWAI (CAN bus) | `driver/twai_general.h` | `esp_driver_twai` (v5.5+, replaces legacy `driver/twai.h`) |

Add component dependencies in `CMakeLists.txt`:
```cmake
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_driver_i2c esp_driver_mcpwm esp_wifi mqtt nvs_flash)
```

## Build Commands (Low Freedom — Use Exactly)

```bash
# Setup (run once per terminal session)
. $IDF_PATH/export.sh

# Set target chip (run once per project)
idf.py set-target esp32    # or esp32s3, esp32c3, esp32c6, esp32h2

# Build, flash, monitor
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor    # Linux
idf.py -p /dev/cu.usbserial-* flash monitor  # macOS
# Ctrl+] to exit monitor

# Configuration
idf.py menuconfig

# Clean builds
idf.py fullclean   # nuclear option — deletes build dir
idf.py reconfigure # re-runs CMake without full clean

# Partition & NVS
idf.py partition-table flash
idf.py erase-flash  # factory reset — erases everything
```

## FreeRTOS Task Architecture

### Task Creation with Core Pinning
```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

// Pin to specific core (0 or 1), or tskNO_AFFINITY for either
xTaskCreatePinnedToCore(sensor_task, "sensor", 4096, NULL, 5, &sensor_handle, 0);
xTaskCreatePinnedToCore(comms_task, "comms", 8192, NULL, 3, &comms_handle, 1);
// Args: function, name, stack_bytes, param, priority(higher=more), handle, core
```

### Inter-Task Communication
```c
#include "freertos/queue.h"
#include "freertos/semphr.h"

// Queue: typed data between tasks
QueueHandle_t data_queue = xQueueCreate(10, sizeof(sensor_data_t));
xQueueSend(data_queue, &reading, pdMS_TO_TICKS(100));     // sender
xQueueReceive(data_queue, &reading, portMAX_DELAY);        // receiver

// Mutex: protect shared resource
SemaphoreHandle_t spi_mutex = xSemaphoreCreateMutex();
if (xSemaphoreTake(spi_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
    // access shared resource
    xSemaphoreGive(spi_mutex);
}

// Event Group: synchronize multiple conditions
#include "freertos/event_groups.h"
EventGroupHandle_t wifi_events = xEventGroupCreate();
#define WIFI_CONNECTED_BIT BIT0
#define MQTT_CONNECTED_BIT BIT1
xEventGroupSetBits(wifi_events, WIFI_CONNECTED_BIT);                     // signal
xEventGroupWaitBits(wifi_events, WIFI_CONNECTED_BIT | MQTT_CONNECTED_BIT,
                    pdFALSE, pdTRUE, portMAX_DELAY);                     // wait for both
```

### Task Watchdog
The Task Watchdog Timer (TWDT) is enabled by default on the idle task of each core. If your task blocks the idle task (e.g., tight loops without `vTaskDelay`), you'll get TWDT panics. Either:
- Add `vTaskDelay(pdMS_TO_TICKS(10))` in loops
- Subscribe your task: `esp_task_wdt_add(NULL)` then periodically `esp_task_wdt_reset()`

## Standard app_main Pattern
```c
#include "esp_log.h"
#include "nvs_flash.h"

static const char *TAG = "main";

void app_main(void)
{
    // 1. Initialize NVS (required before WiFi)
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 2. Initialize peripherals
    // 3. Initialize WiFi/networking
    // 4. Launch tasks
    
    ESP_LOGI(TAG, "System initialized");
    // app_main returns — FreeRTOS scheduler continues running tasks
}
```

## Error Handling Pattern
```c
// Always wrap ESP-IDF calls — ESP_ERROR_CHECK aborts on failure
ESP_ERROR_CHECK(some_init_function());

// For non-fatal operations, check and log
esp_err_t err = i2c_master_transmit(dev, data, len, -1);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "I2C transmit failed: %s", esp_err_to_name(err));
    // handle error gracefully
}
```

## Logging
```c
ESP_LOGE(TAG, "Error: %s", msg);    // Red — errors
ESP_LOGW(TAG, "Warning: %s", msg);  // Yellow — warnings  
ESP_LOGI(TAG, "Info: %s", msg);     // Green — normal info
ESP_LOGD(TAG, "Debug: %s", msg);    // (hidden by default)
ESP_LOGV(TAG, "Verbose: %s", msg);  // (hidden by default)
// Set level: esp_log_level_set("my_tag", ESP_LOG_DEBUG);
// Or globally via menuconfig: Component config → Log → Default log verbosity
```

## Common Gotchas

1. **Stack overflow** — Default 4096 bytes is tight. Bump to 8192+ for tasks doing string formatting, JSON, or TLS. Symptoms: random crashes, corrupted data, guru meditation errors.
2. **I2C "CONFLICT" error** — You're mixing old (`driver/i2c.h`) and new (`driver/i2c_master.h`) APIs. Pick one. On v5.4+ use only the new API.
3. **WiFi needs NVS** — Always call `nvs_flash_init()` before any WiFi functions.
4. **`app_main` is not a task loop** — It runs once and returns. Launch FreeRTOS tasks for continuous work.
5. **Brownout on USB power** — WiFi TX draws ~300mA peaks. If you see "Brownout detector was triggered", use a powered USB hub or external 5V supply.
6. **MQTT config struct changed in v5.x** — The old flat `.uri` field is gone. Use `.broker.address.uri` instead (nested struct).
7. **Component not found** — If you get "undefined reference" linking errors, you probably need to add the component to `REQUIRES` or `PRIV_REQUIRES` in your CMakeLists.txt.
8. **`idf.py menuconfig` won't open** — Needs a terminal that supports ncurses. In VS Code, use the ESP-IDF extension's GUI menuconfig instead.
9. **GCC 14 warnings break builds (v5.4+)** — v5.4 upgraded to GCC 14.2 which adds new warnings. If porting older code with `-Werror`, set `CONFIG_COMPILER_DISABLE_GCC14_WARNINGS=y` in sdkconfig as a temporary workaround.
10. **TLS connections fail silently (v5.5+)** — `mbedtls_ssl_set_hostname()` is now mandatory before TLS handshake. If your HTTPS/MQTTS connections fail after upgrading, this is likely why.
11. **OTA rollback doesn't work** — Requires `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y` in sdkconfig. Without it, `esp_ota_get_state_partition()` won't report `PENDING_VERIFY`.
12. **ADC_ATTEN_DB_11 renamed** — Renamed to `ADC_ATTEN_DB_12` in v5.2. Old name still compiles with a warning but is removed in v6.0.

## Reference Files

For detailed API patterns and code examples, consult these reference files:
- **[references/drivers.md](references/drivers.md)** — I2C master, MCPWM servo/motor, ADC, GPIO, LEDC PWM initialization and usage patterns with complete working code
- **[references/networking.md](references/networking.md)** — WiFi STA connection with reconnection, MQTT client pub/sub, NVS credential storage, HTTP client
- **[references/project-structure.md](references/project-structure.md)** — CMakeLists.txt templates, multi-component project layout, partition tables, OTA setup, managed components via idf_component.yml

Read the appropriate reference file when you need specific driver initialization code or project configuration details.
