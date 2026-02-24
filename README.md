# ESP-IDF Dev Plugin for Claude Code

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that gives Claude deep knowledge of ESP-IDF v5.4/5.5 APIs, drivers, networking, FreeRTOS patterns, and the CMake build system.

## What It Does

Claude automatically activates this skill when it detects ESP-IDF project context — no slash command needed. It provides:

- **New driver APIs** — I2C master, MCPWM (servo/motor), ADC, GPIO, LEDC PWM, UART, SPI, GPTimer, RMT, PCNT, TWAI/CAN
- **FreeRTOS patterns** — Task creation with core pinning, queues, mutexes, event groups, watchdog handling
- **WiFi/MQTT networking** — Station connection with reconnection, MQTT v5.x config, HTTP client, NVS storage, mDNS
- **Build system** — CMakeLists.txt templates, multi-component projects, managed components, partition tables
- **OTA updates** — HTTPS OTA with rollback protection
- **12 documented gotchas** — Stack overflow, I2C API conflicts, GCC 14 warnings, TLS hostname requirements, and more

## Installation

Requires [Claude Code](https://docs.anthropic.com/en/docs/claude-code) v1.0.33+.

```
/plugin marketplace add dropbop/esp-idf-dev-plugin
/plugin install esp-idf-dev@esp-idf-dev-plugin
```

Restart Claude Code after installing. The skill will appear as `esp-idf-dev:esp-idf-dev` in the system prompt.

## When It Triggers

Claude uses this skill automatically when it sees:

- `idf.py`, `esp_err_t`, `ESP_LOG` macros, `xTaskCreate`, or any ESP-IDF API
- ESP-IDF project files (`sdkconfig`, `CMakeLists.txt` with `idf_component_register`)
- ESP32 firmware development context

**Does NOT trigger for** Arduino framework code or ESPHome YAML configs.

## What's Inside

| File | Content |
|------|---------|
| `skills/esp-idf-dev/SKILL.md` | Core reference — driver table, build commands, FreeRTOS, app_main pattern, error handling, gotchas |
| `skills/esp-idf-dev/references/drivers.md` | Complete I2C, MCPWM, ADC, GPIO, LEDC, UART, GPTimer, SPI initialization and usage code |
| `skills/esp-idf-dev/references/networking.md` | WiFi STA, MQTT client, HTTP client, NVS, mDNS patterns with full working examples |
| `skills/esp-idf-dev/references/project-structure.md` | CMake templates, multi-component layout, partition tables, OTA, menuconfig settings |

## ESP-IDF Version Coverage

Targets **ESP-IDF v5.4 and v5.5**. Documents breaking changes from v4.x (MQTT config struct, ADC attenuation rename, GCC 14 warnings) and flags APIs removed in v6.0.

## License

Unlicense
