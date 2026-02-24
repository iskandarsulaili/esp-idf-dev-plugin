# ESP-IDF v5.x Networking Reference

## Table of Contents
- [WiFi Station (STA)](#wifi-station)
- [MQTT Client](#mqtt-client)
- [HTTP Client](#http-client)
- [NVS (Non-Volatile Storage)](#nvs)
- [mDNS](#mdns)
- [Full WiFi + MQTT Integration Pattern](#full-integration-pattern)

---

## WiFi Station

**Headers & Dependencies:**
```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"
```
```cmake
REQUIRES esp_wifi esp_event esp_netif nvs_flash
```

**Connection with Reconnection:**
```c
#include "freertos/event_groups.h"

#define WIFI_SSID      "YourSSID"
#define WIFI_PASS      "YourPassword"
#define MAX_RETRY      5

static EventGroupHandle_t s_wifi_event_group;
#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1
static int s_retry_num = 0;

static void wifi_event_handler(void *arg, esp_event_base_t event_base,
                               int32_t event_id, void *event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_num < MAX_RETRY) {
            esp_wifi_connect();
            s_retry_num++;
            ESP_LOGI(TAG, "Retrying WiFi connection... (%d/%d)", s_retry_num, MAX_RETRY);
        } else {
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
        }
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
        ESP_LOGI(TAG, "Got IP: " IPSTR, IP2STR(&event->ip_info.ip));
        s_retry_num = 0;
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

void wifi_init_sta(void)
{
    s_wifi_event_group = xEventGroupCreate();

    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    // Register event handlers
    esp_event_handler_instance_t instance_any_id;
    esp_event_handler_instance_t instance_got_ip;
    ESP_ERROR_CHECK(esp_event_handler_instance_register(
        WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL, &instance_any_id));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(
        IP_EVENT, IP_EVENT_STA_GOT_IP, &wifi_event_handler, NULL, &instance_got_ip));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    // Block until connected or failed
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
        WIFI_CONNECTED_BIT | WIFI_FAIL_BIT, pdFALSE, pdFALSE, portMAX_DELAY);

    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG, "Connected to WiFi");
    } else if (bits & WIFI_FAIL_BIT) {
        ESP_LOGE(TAG, "Failed to connect to WiFi");
    }
}
```

**Persistent reconnection (production pattern):**
For production, remove `MAX_RETRY` and always reconnect. Add exponential backoff:
```c
} else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
    int delay_ms = (s_retry_num < 10) ? (1000 * (1 << s_retry_num)) : 60000;
    vTaskDelay(pdMS_TO_TICKS(delay_ms));
    esp_wifi_connect();
    s_retry_num++;
}
```

---

## MQTT Client

**Headers & Dependencies:**
```c
#include "mqtt_client.h"
```
```cmake
REQUIRES mqtt
```
Note: In ESP-IDF v6.0+, MQTT is moved to a managed component. Use `idf.py add-dependency espressif/mqtt` and the header becomes `mqtt_client.h` from the external component.

**Basic MQTT Client:**
```c
static esp_mqtt_client_handle_t mqtt_client = NULL;

static void mqtt_event_handler(void *handler_args, esp_event_base_t base,
                               int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;

    switch ((esp_mqtt_event_id_t)event_id) {
    case MQTT_EVENT_CONNECTED:
        ESP_LOGI(TAG, "MQTT connected");
        esp_mqtt_client_subscribe(mqtt_client, "device/cmd", 1);
        // Publish online status
        esp_mqtt_client_publish(mqtt_client, "device/status", "online", 0, 1, true);
        break;

    case MQTT_EVENT_DISCONNECTED:
        ESP_LOGW(TAG, "MQTT disconnected");
        // Auto-reconnect is enabled by default
        break;

    case MQTT_EVENT_DATA:
        ESP_LOGI(TAG, "Topic: %.*s", event->topic_len, event->topic);
        ESP_LOGI(TAG, "Data: %.*s", event->data_len, event->data);
        // Note: topic and data are NOT null-terminated. Use the _len fields.
        break;

    case MQTT_EVENT_ERROR:
        ESP_LOGE(TAG, "MQTT error type: %d", event->error_handle->error_type);
        break;

    default:
        break;
    }
}

void mqtt_app_start(void)
{
    // v5.x config uses nested struct
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtt://192.168.1.100:1883",
        // For TLS:
        // .broker.address.uri = "mqtts://broker.example.com:8883",
        // .broker.verification.certificate = (const char *)server_cert_pem_start,
        // NOTE (v5.5+): mbedtls_ssl_set_hostname() is now mandatory for TLS.
        // The MQTT client handles this automatically when you set the URI,
        // but custom TLS setups must call it explicitly or connections fail.
        .credentials.username = "device01",
        .credentials.authentication.password = "secret",
        .session.last_will = {
            .topic = "device/status",
            .msg = "offline",
            .qos = 1,
            .retain = true,
        },
    };
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(mqtt_client);
}
```

**CRITICAL: v5.x MQTT config struct change.**
The old flat struct (`.uri`, `.username`, `.password`) was replaced:
| Old (v4.x) | New (v5.x) |
|------------|-----------|
| `.uri` | `.broker.address.uri` |
| `.username` | `.credentials.username` |
| `.password` | `.credentials.authentication.password` |
| `.cert_pem` | `.broker.verification.certificate` |
| `.client_cert_pem` | `.credentials.authentication.certificate` |
| `.client_key_pem` | `.credentials.authentication.key` |

**Publishing from another task:**
```c
// esp_mqtt_client_publish is thread-safe
esp_mqtt_client_publish(mqtt_client, "sensors/temp", "23.5", 0, 0, false);
// Args: client, topic, data, data_len (0=use strlen), qos, retain
// Returns msg_id (>0) or -1 on failure
```

---

## HTTP Client

```c
#include "esp_http_client.h"
```
```cmake
REQUIRES esp_http_client
```

```c
esp_http_client_config_t config = {
    .url = "http://api.example.com/data",
    .method = HTTP_METHOD_POST,
};
esp_http_client_handle_t client = esp_http_client_init(&config);

// POST JSON
const char *post_data = "{\"temp\":23.5}";
esp_http_client_set_header(client, "Content-Type", "application/json");
esp_http_client_set_post_field(client, post_data, strlen(post_data));

esp_err_t err = esp_http_client_perform(client);
if (err == ESP_OK) {
    int status = esp_http_client_get_status_code(client);
    int content_len = esp_http_client_get_content_length(client);
    ESP_LOGI(TAG, "HTTP POST status = %d, content_length = %d", status, content_len);
}
esp_http_client_cleanup(client);
```

---

## NVS

Non-volatile storage for key-value pairs. Survives reboot. Used for WiFi credentials, calibration data, settings.

```c
#include "nvs_flash.h"
#include "nvs.h"
```
```cmake
REQUIRES nvs_flash
```

```c
// Init (call once at startup, before WiFi)
esp_err_t ret = nvs_flash_init();
if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    ret = nvs_flash_init();
}
ESP_ERROR_CHECK(ret);

// Write
nvs_handle_t nvs;
ESP_ERROR_CHECK(nvs_open("storage", NVS_READWRITE, &nvs));
ESP_ERROR_CHECK(nvs_set_i32(nvs, "boot_count", 42));
ESP_ERROR_CHECK(nvs_set_str(nvs, "wifi_ssid", "MyNetwork"));
ESP_ERROR_CHECK(nvs_commit(nvs));  // Required to actually write to flash
nvs_close(nvs);

// Read
nvs_handle_t nvs;
ESP_ERROR_CHECK(nvs_open("storage", NVS_READONLY, &nvs));
int32_t boot_count = 0;
nvs_get_i32(nvs, "boot_count", &boot_count);  // Returns ESP_ERR_NVS_NOT_FOUND if key absent

char ssid[33];
size_t ssid_len = sizeof(ssid);
nvs_get_str(nvs, "wifi_ssid", ssid, &ssid_len);  // ssid_len is in/out
nvs_close(nvs);

// Blob (binary data)
ESP_ERROR_CHECK(nvs_set_blob(nvs, "cal_data", cal_buffer, cal_size));
size_t required_size;
nvs_get_blob(nvs, "cal_data", NULL, &required_size);  // get size first
nvs_get_blob(nvs, "cal_data", buffer, &required_size); // then read
```

---

## mDNS

Advertise your device on the local network as `mydevice.local`.

```c
#include "mdns.h"
```
```cmake
REQUIRES mdns
```

```c
ESP_ERROR_CHECK(mdns_init());
ESP_ERROR_CHECK(mdns_hostname_set("mydevice"));
ESP_ERROR_CHECK(mdns_instance_name_set("My ESP32 Sensor"));
// Now reachable at http://mydevice.local
```

---

## Full Integration Pattern

Typical startup sequence for a WiFi + MQTT sensor node:

```c
void app_main(void)
{
    // 1. NVS (required before WiFi)
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 2. Peripherals (I2C bus, sensors, etc.)
    i2c_bus_init();
    sensor_init();

    // 3. WiFi (blocks until connected)
    wifi_init_sta();

    // 4. MQTT (non-blocking, reconnects automatically)
    mqtt_app_start();

    // 5. Launch worker tasks
    xTaskCreatePinnedToCore(sensor_read_task, "sensor", 4096, NULL, 5, NULL, 0);
    xTaskCreatePinnedToCore(mqtt_publish_task, "publish", 4096, NULL, 3, NULL, 1);

    // app_main returns, scheduler keeps running
}
```

**Sensor read task → MQTT publish task via queue:**
```c
static QueueHandle_t sensor_queue;

void sensor_read_task(void *param) {
    sensor_data_t data;
    while (1) {
        data.temperature = read_temperature();
        data.humidity = read_humidity();
        data.timestamp = esp_timer_get_time() / 1000;  // ms
        xQueueSend(sensor_queue, &data, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

void mqtt_publish_task(void *param) {
    sensor_data_t data;
    char json_buf[128];
    while (1) {
        if (xQueueReceive(sensor_queue, &data, portMAX_DELAY) == pdTRUE) {
            snprintf(json_buf, sizeof(json_buf),
                "{\"temp\":%.1f,\"hum\":%.1f,\"ts\":%lld}",
                data.temperature, data.humidity, data.timestamp);
            esp_mqtt_client_publish(mqtt_client, "sensors/data", json_buf, 0, 0, false);
        }
    }
}
```
