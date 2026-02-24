# ESP-IDF v5.x Peripheral Driver Reference

## Table of Contents
- [I2C Master](#i2c-master)
- [MCPWM (Servo & Motor Control)](#mcpwm-servo--motor-control)
- [ADC (Oneshot & Continuous)](#adc)
- [GPIO](#gpio)
- [LEDC (General PWM)](#ledc-general-pwm)
- [UART](#uart)
- [GPTimer (General Purpose Timer)](#gptimer)
- [SPI Master](#spi-master)

---

## I2C Master

The new I2C driver uses a bus-device model: first create the bus, then add devices to it.

**Headers & Dependencies:**
```c
#include "driver/i2c_master.h"
```
```cmake
REQUIRES esp_driver_i2c
```

**Bus + Device Init:**
```c
// 1. Configure and create the I2C bus
i2c_master_bus_handle_t bus_handle;
i2c_master_bus_config_t bus_config = {
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .i2c_port = I2C_NUM_0,
    .scl_io_num = GPIO_NUM_22,
    .sda_io_num = GPIO_NUM_21,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true,  // OK for prototyping; use external 4.7k for production
};
ESP_ERROR_CHECK(i2c_new_master_bus(&bus_config, &bus_handle));

// 2. Add a device on the bus
i2c_master_dev_handle_t dev_handle;
i2c_device_config_t dev_config = {
    .dev_addr_length = I2C_ADDR_BIT_LEN_7,
    .device_address = 0x68,   // e.g., MPU6050 or SCD4x
    .scl_speed_hz = 100000,   // 100kHz standard; 400kHz fast mode
};
ESP_ERROR_CHECK(i2c_master_bus_add_device(bus_handle, &dev_config, &dev_handle));
```

**Read/Write Operations:**
```c
// Write: transmit buffer directly (address handled by driver)
uint8_t write_buf[] = {0x6B, 0x00};  // register addr + data
ESP_ERROR_CHECK(i2c_master_transmit(dev_handle, write_buf, sizeof(write_buf), -1));

// Read register: transmit register address, then receive data
uint8_t reg_addr = 0x75;   // WHO_AM_I register
uint8_t read_buf[1];
ESP_ERROR_CHECK(i2c_master_transmit_receive(dev_handle, &reg_addr, 1, read_buf, 1, -1));
// Last arg: timeout in ms. -1 = use default from device config

// Receive only (device sends data after address ACK)
uint8_t data[6];
ESP_ERROR_CHECK(i2c_master_receive(dev_handle, data, sizeof(data), -1));

// Probe: check if device exists on bus
esp_err_t ret = i2c_master_probe(bus_handle, 0x68, 100);
if (ret == ESP_OK) {
    ESP_LOGI(TAG, "Device found at 0x68");
} else if (ret == ESP_ERR_NOT_FOUND) {
    ESP_LOGW(TAG, "Device not found");
}
```

**Writing to a specific register (common pattern):**
The new API doesn't have a separate "write register" function. Prepend the register address to your data buffer:
```c
// Write value 0x00 to register 0x6B
uint8_t buf[2] = {0x6B, 0x00};
i2c_master_transmit(dev_handle, buf, 2, -1);

// For multi-byte writes, build the buffer:
uint8_t config_buf[3] = {reg_addr, data_byte1, data_byte2};
i2c_master_transmit(dev_handle, config_buf, 3, -1);
```

**Cleanup:**
```c
i2c_master_bus_rm_device(dev_handle);
i2c_del_master_bus(bus_handle);
```

---

## MCPWM (Servo & Motor Control)

MCPWM is a hardware peripheral for precision PWM — ideal for servos and motor drivers. Uses a chain: Timer → Operator → Comparator → Generator.

**Headers & Dependencies:**
```c
#include "driver/mcpwm_prelude.h"
// This single include gives you mcpwm_timer, mcpwm_oper, mcpwm_cmpr, mcpwm_gen
```
```cmake
REQUIRES esp_driver_mcpwm
```

**RC Servo Control (50Hz PWM, 500-2500μs pulse):**
```c
#define SERVO_GPIO          18
#define SERVO_FREQ_HZ       50
#define SERVO_RESOLUTION_HZ 1000000  // 1MHz, 1μs per tick
#define SERVO_MIN_PULSEWIDTH_US 500
#define SERVO_MAX_PULSEWIDTH_US 2500
#define SERVO_MIN_DEGREE    -90
#define SERVO_MAX_DEGREE    90

// Helper: angle → comparator ticks
static uint32_t angle_to_compare(int angle)
{
    return (angle - SERVO_MIN_DEGREE)
         * (SERVO_MAX_PULSEWIDTH_US - SERVO_MIN_PULSEWIDTH_US)
         / (SERVO_MAX_DEGREE - SERVO_MIN_DEGREE)
         + SERVO_MIN_PULSEWIDTH_US;
}

// 1. Timer
mcpwm_timer_handle_t timer = NULL;
mcpwm_timer_config_t timer_config = {
    .group_id = 0,
    .clk_src = MCPWM_TIMER_CLK_SRC_DEFAULT,
    .resolution_hz = SERVO_RESOLUTION_HZ,
    .period_ticks = SERVO_RESOLUTION_HZ / SERVO_FREQ_HZ,  // 20000 ticks = 20ms
    .count_mode = MCPWM_TIMER_COUNT_MODE_UP,
};
ESP_ERROR_CHECK(mcpwm_new_timer(&timer_config, &timer));

// 2. Operator
mcpwm_oper_handle_t oper = NULL;
mcpwm_operator_config_t oper_config = { .group_id = 0 };
ESP_ERROR_CHECK(mcpwm_new_operator(&oper_config, &oper));
ESP_ERROR_CHECK(mcpwm_operator_connect_timer(oper, timer));

// 3. Comparator
mcpwm_cmpr_handle_t comparator = NULL;
mcpwm_comparator_config_t cmpr_config = {
    .flags.update_cmp_on_tez = true,  // update on timer equal zero
};
ESP_ERROR_CHECK(mcpwm_new_comparator(oper, &cmpr_config, &comparator));

// 4. Generator (connects to GPIO)
mcpwm_gen_handle_t generator = NULL;
mcpwm_generator_config_t gen_config = { .gen_gpio_num = SERVO_GPIO };
ESP_ERROR_CHECK(mcpwm_new_generator(oper, &gen_config, &generator));

// 5. Set actions: HIGH on timer empty, LOW on compare match
ESP_ERROR_CHECK(mcpwm_generator_set_action_on_timer_event(generator,
    MCPWM_GEN_TIMER_EVENT_ACTION(MCPWM_TIMER_DIRECTION_UP,
                                  MCPWM_TIMER_EVENT_EMPTY,
                                  MCPWM_GEN_ACTION_HIGH)));
ESP_ERROR_CHECK(mcpwm_generator_set_action_on_compare_event(generator,
    MCPWM_GEN_COMPARE_EVENT_ACTION(MCPWM_TIMER_DIRECTION_UP,
                                    comparator,
                                    MCPWM_GEN_ACTION_LOW)));

// 6. Start
ESP_ERROR_CHECK(mcpwm_timer_enable(timer));
ESP_ERROR_CHECK(mcpwm_timer_start_stop(timer, MCPWM_TIMER_START_NO_STOP));

// Move servo to angle:
ESP_ERROR_CHECK(mcpwm_comparator_set_compare_value(comparator, angle_to_compare(0)));
vTaskDelay(pdMS_TO_TICKS(500));  // wait for servo to reach position
```

**Brushed DC Motor (H-Bridge, e.g., L298N):**
Use two generators on one operator for forward/reverse, or use LEDC for simple single-direction speed control.

---

## ADC

**Oneshot Mode (single conversions, e.g., reading a potentiometer):**
```c
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"
```
```cmake
REQUIRES esp_adc
```

```c
// Init
adc_oneshot_unit_handle_t adc_handle;
adc_oneshot_unit_init_cfg_t init_config = { .unit_id = ADC_UNIT_1 };
ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_config, &adc_handle));

adc_oneshot_chan_cfg_t chan_config = {
    .bitwidth = ADC_BITWIDTH_DEFAULT,
    .atten = ADC_ATTEN_DB_12,  // 0-3.3V range (ESP32); use DB_12 for full range
    // Note: was called ADC_ATTEN_DB_11 before v5.2. Old name still compiles but deprecated.
};
ESP_ERROR_CHECK(adc_oneshot_config_channel(adc_handle, ADC_CHANNEL_6, &chan_config));
// ADC_CHANNEL_6 = GPIO34 on ESP32; check pinmap for your chip variant

// Read
int raw_value;
ESP_ERROR_CHECK(adc_oneshot_read(adc_handle, ADC_CHANNEL_6, &raw_value));

// Optional: calibrated voltage (requires eFuse calibration data)
// NOTE: Calibration scheme is CHIP-SPECIFIC:
//   ESP32, ESP32-C2:  adc_cali_create_scheme_line_fitting()
//   ESP32-S2/S3/C3/C6/H2: adc_cali_create_scheme_curve_fitting()
// Use compile-time guards:
#if ADC_CALI_SCHEME_LINE_FITTING_SUPPORTED
adc_cali_handle_t cali_handle = NULL;
adc_cali_line_fitting_config_t cali_config = {
    .unit_id = ADC_UNIT_1,
    .atten = ADC_ATTEN_DB_12,
    .bitwidth = ADC_BITWIDTH_DEFAULT,
};
if (adc_cali_create_scheme_line_fitting(&cali_config, &cali_handle) == ESP_OK) {
    int voltage_mv;
    adc_cali_raw_to_voltage(cali_handle, raw_value, &voltage_mv);
    ESP_LOGI(TAG, "Voltage: %d mV", voltage_mv);
}
#elif ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED
adc_cali_handle_t cali_handle = NULL;
adc_cali_curve_fitting_config_t cali_config = {
    .unit_id = ADC_UNIT_1,
    .atten = ADC_ATTEN_DB_12,
    .bitwidth = ADC_BITWIDTH_DEFAULT,
};
if (adc_cali_create_scheme_curve_fitting(&cali_config, &cali_handle) == ESP_OK) {
    int voltage_mv;
    adc_cali_raw_to_voltage(cali_handle, raw_value, &voltage_mv);
    ESP_LOGI(TAG, "Voltage: %d mV", voltage_mv);
}
#endif
```

---

## GPIO

```c
#include "driver/gpio.h"
```
```cmake
REQUIRES esp_driver_gpio
```

```c
// Output
gpio_config_t out_conf = {
    .pin_bit_mask = (1ULL << GPIO_NUM_2),
    .mode = GPIO_MODE_OUTPUT,
    .pull_up_en = GPIO_PULLUP_DISABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type = GPIO_INTR_DISABLE,
};
gpio_config(&out_conf);
gpio_set_level(GPIO_NUM_2, 1);

// Input with interrupt
gpio_config_t in_conf = {
    .pin_bit_mask = (1ULL << GPIO_NUM_4),
    .mode = GPIO_MODE_INPUT,
    .pull_up_en = GPIO_PULLUP_ENABLE,
    .intr_type = GPIO_INTR_NEGEDGE,
};
gpio_config(&in_conf);
gpio_install_isr_service(0);
gpio_isr_handler_add(GPIO_NUM_4, gpio_isr_handler, (void*)GPIO_NUM_4);

// ISR handler — keep it minimal, signal a task instead
static void IRAM_ATTR gpio_isr_handler(void *arg)
{
    uint32_t gpio_num = (uint32_t)arg;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, NULL);
}
```

---

## LEDC (General PWM)

For simple PWM (LEDs, fans, buzzers). Easier than MCPWM but no dead-time or capture.

```c
#include "driver/ledc.h"
```
```cmake
REQUIRES esp_driver_ledc
```

```c
ledc_timer_config_t timer_conf = {
    .speed_mode = LEDC_LOW_SPEED_MODE,
    .duty_resolution = LEDC_TIMER_13_BIT,  // 0-8191
    .timer_num = LEDC_TIMER_0,
    .freq_hz = 5000,
    .clk_cfg = LEDC_AUTO_CLK,
};
ledc_timer_config(&timer_conf);

ledc_channel_config_t ch_conf = {
    .speed_mode = LEDC_LOW_SPEED_MODE,
    .channel = LEDC_CHANNEL_0,
    .timer_sel = LEDC_TIMER_0,
    .gpio_num = GPIO_NUM_2,
    .duty = 0,
    .hpoint = 0,
};
ledc_channel_config(&ch_conf);

// Set duty (0 to 2^resolution - 1)
ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 4096);  // ~50%
ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);

// Hardware fade
ledc_fade_func_install(0);
ledc_set_fade_with_time(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 8191, 2000);
ledc_fade_start(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, LEDC_FADE_NO_WAIT);
```

---

## UART

```c
#include "driver/uart.h"
```
```cmake
REQUIRES esp_driver_uart
```

```c
const uart_port_t uart_num = UART_NUM_1;
uart_config_t uart_config = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
    .parity = UART_PARITY_DISABLE,
    .stop_bits = UART_STOP_BITS_1,
    .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
    .source_clk = UART_SCLK_DEFAULT,
};
ESP_ERROR_CHECK(uart_param_config(uart_num, &uart_config));
ESP_ERROR_CHECK(uart_set_pin(uart_num, 17, 16, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE));
// TX=17, RX=16

const int buf_size = 1024;
ESP_ERROR_CHECK(uart_driver_install(uart_num, buf_size * 2, 0, 0, NULL, 0));

// Write
const char *msg = "Hello\n";
uart_write_bytes(uart_num, msg, strlen(msg));

// Read (blocks until data or timeout)
uint8_t data[128];
int len = uart_read_bytes(uart_num, data, sizeof(data), pdMS_TO_TICKS(100));
```

---

## GPTimer

```c
#include "driver/gptimer.h"
```
```cmake
REQUIRES esp_driver_gptimer
```

```c
gptimer_handle_t gptimer = NULL;
gptimer_config_t timer_config = {
    .clk_src = GPTIMER_CLK_SRC_DEFAULT,
    .direction = GPTIMER_COUNT_UP,
    .resolution_hz = 1000000,  // 1MHz = 1μs per tick
};
ESP_ERROR_CHECK(gptimer_new_timer(&timer_config, &gptimer));

// Alarm callback
gptimer_alarm_config_t alarm_config = {
    .alarm_count = 1000000,    // 1 second
    .flags.auto_reload_on_alarm = true,
    .reload_count = 0,
};
gptimer_event_callbacks_t cbs = { .on_alarm = timer_alarm_cb };
gptimer_register_event_callbacks(gptimer, &cbs, NULL);
gptimer_set_alarm_action(gptimer, &alarm_config);
gptimer_enable(gptimer);
gptimer_start(gptimer);

// Callback (runs in ISR context — keep fast, signal a task)
static bool IRAM_ATTR timer_alarm_cb(gptimer_handle_t timer,
    const gptimer_alarm_event_data_t *edata, void *user_data)
{
    BaseType_t high_task_wakeup = pdFALSE;
    // Signal task via queue, notification, etc.
    return high_task_wakeup == pdTRUE;
}
```

---

## SPI Master

```c
#include "driver/spi_master.h"
```
```cmake
REQUIRES esp_driver_spi
```

```c
spi_bus_config_t bus_cfg = {
    .miso_io_num = GPIO_NUM_19,
    .mosi_io_num = GPIO_NUM_23,
    .sclk_io_num = GPIO_NUM_18,
    .quadwp_io_num = -1,
    .quadhd_io_num = -1,
    .max_transfer_sz = 4096,
};
ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &bus_cfg, SPI_DMA_CH_AUTO));

spi_device_handle_t spi_dev;
spi_device_interface_config_t dev_cfg = {
    .clock_speed_hz = 1 * 1000 * 1000,  // 1 MHz
    .mode = 0,                            // CPOL=0, CPHA=0
    .spics_io_num = GPIO_NUM_5,
    .queue_size = 7,
};
ESP_ERROR_CHECK(spi_bus_add_device(SPI2_HOST, &dev_cfg, &spi_dev));

// Transaction
spi_transaction_t t = {
    .length = 8 * 2,          // bits
    .tx_buffer = tx_data,
    .rx_buffer = rx_data,
};
ESP_ERROR_CHECK(spi_device_transmit(spi_dev, &t));
```
