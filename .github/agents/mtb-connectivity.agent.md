---
name: mtb-connectivity
description: Add network connectivity to ModusToolbox projects — WiFi STA, MQTT pub/sub, HTTP client, TLS, BLE Central (GATT client), library dependency management, JSON parsing, and reconnection strategies. Use for any networking, cloud, IoT, BLE, or REST API integration task.
tools: ["read", "edit", "create", "search", "shell"]
---

# Network Connectivity — WiFi, MQTT, HTTP for ModusToolbox™

You are an expert in adding network connectivity to ModusToolbox™ projects. You handle WiFi STA connections, MQTT publish/subscribe, HTTP client requests, TLS configuration, library dependency management, and robust reconnection strategies.

> **Applies to:** PSOC Edge (CM33 NS), PSOC 6, and any kit with CYW43xxx/CYW55xxx WiFi radio.
> **Prerequisites:** Project must have been created via `project-creator-cli` (see `mtb-project` agent). Read `CONTEXT.md` for target device.
> **Deep-dive:** `reference/middleware-compatibility.md` for library version compatibility and super-dependency bundles.

---

# Part 1: Library Dependency Management

## Adding Libraries to a ModusToolbox Project

1. **Identify the library** at `https://github.com/Infineon/[library-name]`
   - Master index: `https://github.com/Infineon/modustoolbox-software/blob/master/README.md`

2. **Read the library README** for required `COMPONENTS+=` and `DEFINES+=`

3. **Determine `deps/` location** (from `CONTEXT.md`):
   - PSOC Edge: `proj_cm33_ns/deps/` (or `proj_cm55/deps/` for CM55 libs)
   - All other families: `deps/` at project root

4. **Create `.mtb` file** in the correct `deps/` directory:
   ```
   https://github.com/Infineon/[library-name]#latest-vX.X#$$ASSET_REPO$$/[library-name]/latest-vX.X
   ```

5. **Version tag conventions:**
   - `latest-vX.X` — latest patch within major.minor (recommended)
   - `release-vX.X.X` — pinned to exact version (reproducible builds)

6. **URL formats:**
   - `https://github.com/Infineon/[name]` — standard for Infineon libraries
   - `https://github.com/[org]/[name]` — third-party (e.g., lvgl/lvgl)
   - `mtb://[name]` — manifest-redirected (only for manifest-registered libs)

7. **Run `make getlibs`** after adding all `.mtb` files

8. **Do NOT add manual `SOURCES+=` for libraries with `props.json`** — the MTB build system auto-discovers their sources. Only use manual `SOURCES+=` for custom application files or libraries with non-standard structure. See `mtb-project` agent, "Library Auto-Discovery" section.

9. **If you modify a library, take local ownership** — copy it to `proj_cm33_ns/libs/`, delete `.git`, and add a `.cyignore` entry to exclude the `mtb_shared` original. `make getlibs` overwrites `mtb_shared/` silently. See `mtb-project` agent, Part 2B.

### Common Dependency Chains

| Capability | Required libraries |
|---|---|
| WiFi + MQTT | wifi-connection-manager, mqtt, secure-sockets, lwip, mbedtls, freertos |
| HTTP | http-client, secure-sockets, lwip, mbedtls, freertos |
| BLE | btstack-integration, bluetooth-freertos, freertos |

---

# Part 2: WiFi STA Connection

## Required Libraries

Add these `.mtb` files to `deps/` (or `proj_cm33_ns/deps/` for PSOC Edge):

```
https://github.com/Infineon/wifi-connection-manager#latest-v3.X#$$ASSET_REPO$$/wifi-connection-manager/latest-v3.X
https://github.com/Infineon/mqtt#latest-v4.X#$$ASSET_REPO$$/mqtt/latest-v4.X
https://github.com/Infineon/secure-sockets#latest-v3.X#$$ASSET_REPO$$/secure-sockets/latest-v3.X
https://github.com/Infineon/lwip#latest-v2.X#$$ASSET_REPO$$/lwip/latest-v2.X
https://github.com/Infineon/mbedtls#latest-v3.X#$$ASSET_REPO$$/mbedtls/latest-v3.X
https://github.com/Infineon/freertos#latest-v10.X#$$ASSET_REPO$$/freertos/latest-v10.X
https://github.com/cypresssemiconductorco/retarget-io#latest-v1.X#$$ASSET_REPO$$/retarget-io/latest-v1.X
```

## Required Makefile Configuration

```makefile
COMPONENTS+=FREERTOS LWIP MBEDTLS
DEFINES+=CYBSP_WIFI_CAPABLE
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF
```

## WiFi Connection Pattern

```c
#include "cy_wcm.h"

#define WIFI_SSID       "YOUR_SSID"
#define WIFI_PASSWORD   "YOUR_PASSWORD"
#define WIFI_SECURITY   CY_WCM_SECURITY_WPA2_AES_PSK
#define MAX_WIFI_RETRIES    5
#define WIFI_RETRY_DELAY_MS 2000

cy_rslt_t wifi_connect(void)
{
    cy_wcm_config_t wcm_config = { .interface = CY_WCM_INTERFACE_TYPE_STA };
    cy_rslt_t result = cy_wcm_init(&wcm_config);
    if (result != CY_RSLT_SUCCESS) return result;

    cy_wcm_connect_params_t connect_params = {0};
    memcpy(connect_params.ap_credentials.SSID, WIFI_SSID, strlen(WIFI_SSID));
    memcpy(connect_params.ap_credentials.password, WIFI_PASSWORD, strlen(WIFI_PASSWORD));
    connect_params.ap_credentials.security = WIFI_SECURITY;

    cy_wcm_ip_address_t ip_addr;
    for (int retry = 0; retry < MAX_WIFI_RETRIES; retry++)
    {
        result = cy_wcm_connect_ap(&connect_params, &ip_addr);
        if (result == CY_RSLT_SUCCESS)
        {
            printf("WiFi connected. IP: %lu.%lu.%lu.%lu\n",
                   (ip_addr.ip.v4 & 0xFF), (ip_addr.ip.v4 >> 8) & 0xFF,
                   (ip_addr.ip.v4 >> 16) & 0xFF, (ip_addr.ip.v4 >> 24) & 0xFF);
            return CY_RSLT_SUCCESS;
        }
        printf("WiFi connect failed (attempt %d/%d)\n", retry + 1, MAX_WIFI_RETRIES);
        vTaskDelay(pdMS_TO_TICKS(WIFI_RETRY_DELAY_MS));
    }
    return result;
}
```

---

# Part 2B: PSOC Edge WiFi — Platform-Specific Requirements

> **Applies to:** PSOC Edge E84 only. PSOC 6 does not need these steps.

## SDIO Initialization Before cy_wcm_init()

On PSOC 6, `cy_wcm_init()` handles WiFi hardware init internally. **On PSOC Edge E84, the SDIO bus to the CYW55500 radio must be explicitly initialized first.** Without it, `cy_wcm_init()` hangs indefinitely.

**Critical ordering:**
```c
/* 1. Zero the config struct FIRST */
cy_wcm_config_t wcm_config;
memset(&wcm_config, 0, sizeof(wcm_config));
wcm_config.interface = CY_WCM_INTERFACE_TYPE_STA;

/* 2. Initialize SDIO bus to the radio */
app_sdio_init();  /* Sets up SDIO HAL, GPIO pins, interrupt handlers */

/* 3. Pass the SDIO instance to WCM */
wcm_config.wifi_interface_instance = &sdio_instance;

/* 4. NOW init WCM */
cy_rslt_t result = cy_wcm_init(&wcm_config);
```

Copy the `app_sdio_init()` pattern from a working WiFi example (e.g., `wifi_task.c` in `mtb-example-psoc-edge-btstack-wifi-onboarding`).

## CYW55500 Reset Override

The CYW55500 radio requires a different reset sequence than CYW43xxx. Override the weak BSP function:

```c
void _cybsp_wifi_reset_wifi_chip(void)
{
    cyhal_gpio_write(CYBSP_WIFI_WL_REG_ON, 0);
    Cy_SysLib_Delay(10);
    cyhal_gpio_write(CYBSP_WIFI_WL_REG_ON, 1);
    Cy_SysLib_Delay(50);  /* CYW55500 needs 50ms settling */
}
```

## FreeRTOS Heap for WiFi + MQTT + TLS

The default `configTOTAL_HEAP_SIZE` of 64 KB is **not enough** for WiFi + MQTT + TLS on CM33. Minimum 128 KB required:

| Component | Approx Heap |
|-----------|------------|
| WiFi driver (CYW55500) | ~18 KB |
| lwIP | ~16 KB |
| mbedTLS (TLS 1.2) | ~40 KB |
| MQTT + FreeRTOS overhead | ~18 KB |
| **Total** | **~92 KB** |

```c
/* FreeRTOSConfig.h */
#define configTOTAL_HEAP_SIZE    ((size_t)(128 * 1024))
```

**Failure mode:** `pvPortMalloc()` returns NULL during TLS handshake → mbedTLS dereferences null → HardFault. Non-obvious.

## SRAM Partitioning for Heavy Connectivity Stacks

Default Device Configurator gives CM33 256 KB data SRAM. WiFi + MQTT + TLS + BLE can push utilization to 98%. If WiFi buffer allocation fails intermittently, increase CM33 data SRAM to at least 384–512 KB via Device Configurator Memory Configuration.

---

# Part 3: MQTT Connection + Publish/Subscribe

## MQTT Connect + Publish

```c
#include "cy_mqtt_api.h"

#define MQTT_BROKER_URL     "mqtt.example.com"
#define MQTT_BROKER_PORT    8883
#define MQTT_CLIENT_ID      "psoc-edge-device-01"
#define MQTT_QOS            CY_MQTT_QOS1

static cy_mqtt_t mqtt_handle;

static const char root_ca_cert[] =
    "-----BEGIN CERTIFICATE-----\n"
    "... your broker's root CA here ...\n"
    "-----END CERTIFICATE-----\n";

cy_rslt_t mqtt_connect(void)
{
    cy_rslt_t result = cy_mqtt_init();
    if (result != CY_RSLT_SUCCESS) return result;

    cy_mqtt_broker_info_t broker_info = {
        .hostname = MQTT_BROKER_URL, .hostname_len = strlen(MQTT_BROKER_URL),
        .port = MQTT_BROKER_PORT,
    };
    cy_awsport_ssl_credentials_t tls_creds = {
        .root_ca = root_ca_cert, .root_ca_size = sizeof(root_ca_cert),
    };

    result = cy_mqtt_create(NULL, NULL, NULL, &broker_info, &tls_creds, &mqtt_handle);
    if (result != CY_RSLT_SUCCESS) return result;

    cy_mqtt_connect_info_t conn_info = {
        .client_id = MQTT_CLIENT_ID, .client_id_len = strlen(MQTT_CLIENT_ID),
        .keep_alive_sec = 60, .clean_session = true,
    };
    return cy_mqtt_connect(mqtt_handle, &conn_info);
}

cy_rslt_t mqtt_publish(const char *topic, const char *payload)
{
    cy_mqtt_publish_info_t pub_info = {
        .qos = MQTT_QOS, .topic = topic, .topic_len = strlen(topic),
        .payload = payload, .payload_len = strlen(payload),
    };
    return cy_mqtt_publish(mqtt_handle, &pub_info);
}
```

## MQTT Subscribe + Callback

```c
void mqtt_event_callback(cy_mqtt_t handle, cy_mqtt_event_t event, void *user_data)
{
    switch (event.type)
    {
        case CY_MQTT_EVENT_TYPE_PUBLISH_RECEIVE:
        {
            cy_mqtt_publish_info_t *msg = &event.data.pub_msg.received_message;
            printf("MQTT RX [%.*s]: %.*s\n",
                   (int)msg->topic_len, msg->topic,
                   (int)msg->payload_len, msg->payload);
            break;
        }
        case CY_MQTT_EVENT_TYPE_DISCONNECT:
            printf("MQTT disconnected — schedule reconnect\n");
            break;
        default:
            break;
    }
}

cy_rslt_t mqtt_subscribe(const char *topic)
{
    cy_mqtt_subscribe_info_t sub_info = {
        .qos = MQTT_QOS, .topic = topic, .topic_len = strlen(topic),
    };
    return cy_mqtt_subscribe(mqtt_handle, &sub_info, 1);
}
```

---

# Part 4: HTTP Client

## HTTP GET — Complete Pattern (Port 80)

```c
#include "cy_secure_sockets.h"

#define HTTP_PORT            (80U)
#define HTTP_BUF_SIZE        (4096U)
#define SOCKET_TIMEOUT_MS    (10000U)
#define RECV_CHUNK_SIZE      (1460U)

static char response_buf[HTTP_BUF_SIZE];

size_t http_get(const char *hostname, const char *path, char **body_out)
{
    cy_rslt_t result;
    cy_socket_t socket_handle;
    *body_out = NULL;

    cy_socket_ip_address_t ip_addr;
    result = cy_socket_gethostbyname(hostname, CY_SOCKET_IP_VER_V4, &ip_addr);
    if (result != CY_RSLT_SUCCESS) return 0;

    result = cy_socket_create(CY_SOCKET_DOMAIN_AF_INET, CY_SOCKET_TYPE_STREAM,
                               CY_SOCKET_IPPROTO_TCP, &socket_handle);
    if (result != CY_RSLT_SUCCESS) return 0;

    uint32_t timeout = SOCKET_TIMEOUT_MS;
    cy_socket_setsockopt(socket_handle, CY_SOCKET_SOL_SOCKET,
                          CY_SOCKET_SO_RCVTIMEO, &timeout, sizeof(timeout));
    cy_socket_setsockopt(socket_handle, CY_SOCKET_SOL_SOCKET,
                          CY_SOCKET_SO_SNDTIMEO, &timeout, sizeof(timeout));

    cy_socket_sockaddr_t addr = { .ip_address = ip_addr, .port = HTTP_PORT };
    result = cy_socket_connect(socket_handle, &addr, sizeof(addr));
    if (result != CY_RSLT_SUCCESS) { cy_socket_delete(socket_handle); return 0; }

    char request[384];
    int req_len = snprintf(request, sizeof(request),
        "GET %s HTTP/1.1\r\nHost: %s\r\nConnection: close\r\nAccept: application/json\r\n\r\n",
        path, hostname);

    uint32_t bytes_sent = 0;
    result = cy_socket_send(socket_handle, request, (uint32_t)req_len,
                             CY_SOCKET_FLAGS_NONE, &bytes_sent);
    if (result != CY_RSLT_SUCCESS) {
        cy_socket_disconnect(socket_handle, SOCKET_TIMEOUT_MS);
        cy_socket_delete(socket_handle); return 0;
    }

    size_t total = 0;
    while (total < (HTTP_BUF_SIZE - 1)) {
        uint32_t chunk = HTTP_BUF_SIZE - 1 - (uint32_t)total;
        if (chunk > RECV_CHUNK_SIZE) chunk = RECV_CHUNK_SIZE;
        uint32_t received = 0;
        result = cy_socket_recv(socket_handle, response_buf + total, chunk,
                                 CY_SOCKET_FLAGS_NONE, &received);
        if (received > 0) total += received;
        if (result != CY_RSLT_SUCCESS) break;
    }
    response_buf[total] = '\0';

    cy_socket_disconnect(socket_handle, SOCKET_TIMEOUT_MS);
    cy_socket_delete(socket_handle);
    if (total == 0) return 0;

    char *body = strstr(response_buf, "\r\n\r\n");
    if (!body) return 0;
    body += 4;
    *body_out = body;
    return total - (size_t)(body - response_buf);
}
```

## JSON Parsing with coreJSON

```c
#include "core_json.h"

float json_get_float(char *json, size_t len, const char *query)
{
    char *value = NULL; size_t value_len = 0;
    if (JSON_Search(json, len, query, strlen(query), &value, &value_len) == JSONSuccess && value_len > 0) {
        char save = value[value_len]; value[value_len] = '\0';
        float result = (float)atof(value); value[value_len] = save;
        return result;
    }
    return 0.0f;
}

bool json_get_string(char *json, size_t len, const char *query, char *out, size_t out_size)
{
    char *value = NULL; size_t value_len = 0;
    if (JSON_Search(json, len, query, strlen(query), &value, &value_len) == JSONSuccess && value_len > 0) {
        const char *src = value; size_t src_len = value_len;
        if (src_len >= 2 && src[0] == '"' && src[src_len - 1] == '"') { src++; src_len -= 2; }
        size_t copy = (src_len < out_size - 1) ? src_len : (out_size - 1);
        memcpy(out, src, copy); out[copy] = '\0';
        return true;
    }
    return false;
}
```

**coreJSON query syntax:** Dots for nested objects (`city.name`), brackets for arrays (`list[0].main.temp`).

## Buffer Sizing Guide

| Payload size | `HTTP_BUF_SIZE` | Notes |
|---|---|---|
| Small JSON (<1 KB) | 2048 | Includes ~300 bytes HTTP headers |
| Medium API response | 4096–6144 | Typical REST API |
| Large payloads | 8192+ | Watch CM33 NS heap (~190 KB on PSOC Edge) |

---

# Part 5: Reconnection Strategy

```c
void network_monitor_task(void *arg)
{
    for (;;) {
        vTaskDelay(pdMS_TO_TICKS(10000));
        if (!cy_wcm_is_connected_to_ap()) {
            printf("WiFi lost — reconnecting...\n");
            wifi_connect();
            mqtt_connect();
        }
    }
}
```

---

# Part 6: Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `CYBSP_WIFI_CAPABLE` define | WiFi init fails silently | Add to Makefile DEFINES |
| Missing `LWIP` or `MBEDTLS` component | Linker errors on cy_wcm/cy_mqtt | Add to Makefile COMPONENTS |
| Non-TLS port (1883) in production | Data in plaintext | Use port 8883 with root CA |
| No reconnection logic | Device goes offline permanently | Add network monitor task |
| Response buffer too small | Truncated JSON, parse failures | Size for max expected response + headers |
| Missing `Connection: close` | recv blocks until timeout | Always include header |
| No socket timeout | Task hangs indefinitely | Set CY_SOCKET_SO_RCVTIMEO |
| Stack-allocated response buffer | Stack overflow > 2 KB | Use pvPortMalloc() |
| DNS failure after WiFi reconnect | Transient gethostbyname failures | Retry with backoff |
| Publishing faster than TLS can encrypt | MQTT queue backlog, disconnect | Rate-limit (1-2/sec typical) |
| PSOC Edge: missing SDIO init before `cy_wcm_init()` | `cy_wcm_init()` hangs indefinitely | See Part 2B — init SDIO HAL first |
| MQTT reconnect without disconnect first | Error `0x08060009` (stale socket) | Call `cy_mqtt_disconnect()` before `cy_mqtt_connect()` even after broker-initiated disconnect |
| FreeRTOS heap too small for WiFi + TLS | HardFault during TLS handshake | `configTOTAL_HEAP_SIZE` ≥ 128 KB |

---

# Part 7: BLE Central (GATT Client)

> **Applies to:** PSOC Edge (CM33 NS) with CYW55xxx combo radio (WiFi + BLE shared).
> **Key constraint:** WiFi and BLE share the same radio firmware on CYW55xxx. BLE HCI transport is only available AFTER the radio firmware is loaded by the WiFi Host Driver (WHD) during `cy_wcm_init()`.

## Required Libraries

Add to `proj_cm33_ns/deps/`:

```
# btstack-integration.mtb
https://github.com/Infineon/btstack-integration#latest-v4.X#$$ASSET_REPO$$/btstack-integration/latest-v4.X

# bluetooth-freertos.mtb
https://github.com/Infineon/bluetooth-freertos#latest-v6.X#$$ASSET_REPO$$/bluetooth-freertos/latest-v6.X
```

> WiFi libraries (wifi-connection-manager, lwip, etc.) are also required since WHD loads the shared radio firmware.

## Required Makefile Configuration

```makefile
COMPONENTS+=FREERTOS LWIP MBEDTLS BTSS
DEFINES+=CYBSP_WIFI_CAPABLE CY_RETARGET_IO_CONVERT_LF_TO_CRLF
```

## Radio Firmware Sharing — WiFi/BLE Coexistence (CRITICAL)

On CYW55xxx (used by KIT_PSE84_EVAL_EPC2), WiFi and BLE share a single radio chip. The BLE HCI transport layer becomes available only after WHD loads the radio firmware during `cy_wcm_init()`.

**Initialization order:**
```
1. cy_wcm_init()          ← loads CYW55xxx firmware (WiFi + BLE combo FW)
2. Signal BLE task         ← radio firmware ready, HCI transport available
3. wiced_bt_stack_init()  ← BLE stack initialization (MUST be after step 1)
```

**Implementation pattern — event-driven readiness:**
```c
/* WiFi task signals BLE task after radio FW loads */
static TaskHandle_t ble_task_handle;

void wifi_task(void *arg)
{
    cy_wcm_config_t cfg = { .interface = CY_WCM_INTERFACE_TYPE_STA };
    cy_wcm_init(&cfg);   /* Radio FW loaded here */

    /* Signal BLE task that HCI transport is ready */
    xTaskNotifyGive(ble_task_handle);

    /* Continue with WiFi connect... */
    wifi_connect();
}

void ble_task(void *arg)
{
    printf("[BLE] Waiting for radio firmware...\n");
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);  /* Block until WiFi loads FW */
    printf("[BLE] Radio ready, initializing btstack\n");

    wiced_bt_stack_init(bt_management_callback, &wiced_bt_cfg_settings);
    /* ... */
}
```

> ⚠️ Calling `wiced_bt_stack_init()` BEFORE `cy_wcm_init()` returns → undefined behavior (HCI transport not initialized). This is the #1 cause of "BLE does nothing" on combo radio kits.

## BLE Central — Scan, Connect, Discover, Subscribe

### Management Callback Structure

```c
wiced_result_t bt_management_callback(wiced_bt_management_evt_t event,
                                       wiced_bt_management_evt_data_t *p_event_data)
{
    switch (event) {
    case BTM_ENABLED_EVT:
        if (p_event_data->enabled.status == WICED_BT_SUCCESS) {
            /* Stack is up — start scanning */
            wiced_bt_ble_scan(BTM_BLE_SCAN_TYPE_HIGH_DUTY, true, scan_result_cb);
        }
        break;
    case BTM_BLE_SCAN_STATE_CHANGED_EVT:
        break;
    default:
        break;
    }
    return WICED_BT_SUCCESS;
}
```

### Scan Callback — Match by Device Name or UUID

```c
static void scan_result_cb(wiced_bt_ble_scan_results_t *p_scan_result,
                            uint8_t *p_adv_data)
{
    if (p_scan_result == NULL) return;  /* Scan complete */

    /* Parse advertisement for local name */
    uint8_t *p_name = NULL;
    uint8_t  name_len = 0;
    p_name = wiced_bt_ble_check_advertising_data(p_adv_data,
                 BTM_BLE_ADVERT_TYPE_NAME_COMPLETE, &name_len);

    if (p_name && name_len > 0 && memcmp(p_name, "MyPeripheral", 12) == 0) {
        /* Found target device — stop scan and connect */
        wiced_bt_ble_scan(BTM_BLE_SCAN_TYPE_NONE, true, scan_result_cb);

        wiced_bt_gatt_le_connect(p_scan_result->remote_bd_addr,
                                  p_scan_result->ble_addr_type,
                                  BLE_CONN_MODE_HIGH_DUTY, true);
    }
}
```

### GATT Event Handler — Discovery + Notification

```c
static wiced_bt_gatt_status_t gatt_callback(wiced_bt_gatt_evt_t event,
                                             wiced_bt_gatt_event_data_t *p_data)
{
    switch (event) {
    case GATT_CONNECTION_STATUS_EVT:
        if (p_data->connection_status.connected) {
            conn_id = p_data->connection_status.conn_id;
            /* Start service discovery */
            wiced_bt_gatt_client_send_discover(conn_id,
                GATT_DISCOVER_SERVICES_ALL, NULL);
        } else {
            /* Disconnected — restart scan after delay */
            conn_id = 0;
            vTaskDelay(pdMS_TO_TICKS(3000));
            wiced_bt_ble_scan(BTM_BLE_SCAN_TYPE_HIGH_DUTY, true, scan_result_cb);
        }
        break;

    case GATT_DISCOVERY_RESULT_EVT:
        handle_discovery_result(p_data);
        break;

    case GATT_DISCOVERY_CPLT_EVT:
        handle_discovery_complete(p_data);
        break;

    case GATT_OPERATION_CPLT_EVT:
        if (p_data->operation_complete.op == GATTC_OPTYPE_NOTIFICATION) {
            handle_notification(p_data);
        }
        break;

    default:
        break;
    }
    return WICED_BT_GATT_SUCCESS;
}
```

### 128-bit Vendor UUID Matching (CRITICAL: Little-Endian Byte Order)

BLE stores 128-bit UUIDs in **little-endian** byte order. When matching a vendor UUID from a spec or phone app (which displays big-endian), you must reverse it:

```c
/* UUID from spec/LightBlue: 3950866D-BE3F-4810-8C78-999E9E58A3A6
 * Stored little-endian for btstack comparison: */
static const uint8_t TARGET_CHAR_UUID_128[] = {
    0xA6, 0xA3, 0x58, 0x9E, 0x9E, 0x99, 0x78, 0x8C,
    0x10, 0x48, 0x3F, 0xBE, 0x6D, 0x86, 0x50, 0x39
};

/* Match during GATT_DISCOVERY_RESULT_EVT: */
if (p_char->char_uuid.len == LEN_UUID_128 &&
    memcmp(p_char->char_uuid.uu.uuid128, TARGET_CHAR_UUID_128, 16) == 0) {
    target_char_handle = p_char->val_handle;
    target_cccd_handle = p_char->val_handle + 1; /* CCCD is typically next handle */
}
```

> ⚠️ **Do NOT use a "first characteristic with NOTIFY property" heuristic** — this often matches Generic Access or other standard characteristics. Always match by exact UUID.

### Enabling Notifications — Static CCCD Write Buffer (CRITICAL)

The CCCD (Client Characteristic Configuration Descriptor) write tells the peripheral to start sending notifications. The write buffer **MUST be static** — btstack reads it asynchronously after the function returns:

```c
void enable_notifications(uint16_t conn_id, uint16_t cccd_handle)
{
    /* MUST be static — btstack reads asynchronously after this function returns */
    static uint8_t cccd_val[] = { GATT_CLIENT_CONFIG_NOTIFICATION & 0xFF,
                                   (GATT_CLIENT_CONFIG_NOTIFICATION >> 8) & 0xFF };
    static wiced_bt_gatt_write_hdr_t write_hdr;

    write_hdr.handle   = cccd_handle;
    write_hdr.offset   = 0;
    write_hdr.len      = 2;
    write_hdr.auth_req = GATT_AUTH_REQ_NONE;

    wiced_bt_gatt_status_t status = wiced_bt_gatt_client_send_write(
        conn_id, GATT_REQ_WRITE, &write_hdr, cccd_val, NULL);

    printf("[BLE] CCCD write (handle 0x%04X): %s\n", cccd_handle,
           status == WICED_BT_GATT_SUCCESS ? "OK" : "FAILED");
}
```

> ⚠️ **Stack-local buffers for CCCD writes = silent failure.** The write appears to succeed but notifications never arrive because btstack reads freed stack memory. This is the #2 most common BLE integration bug.

### Notification Data Handling

```c
static void handle_notification(wiced_bt_gatt_event_data_t *p_data)
{
    wiced_bt_gatt_operation_complete_t *op = &p_data->operation_complete;

    if (op->response_data.att_value.handle == target_char_handle) {
        uint8_t *val = op->response_data.att_value.p_data;
        uint16_t len = op->response_data.att_value.len;

        if (len >= 1) {
            uint8_t sensor_state = val[0];
            /* Process notification data — update shared state, send via IPC, etc. */
        }
    }
}
```

## BLE Central — Reconnection on Disconnect

The peripheral may power-cycle, go out of range, or drop the connection. Always handle `GATT_CONNECTION_STATUS_EVT` with `connected == false`:

```c
case GATT_CONNECTION_STATUS_EVT:
    if (!p_data->connection_status.connected) {
        printf("[BLE] Disconnected (reason: 0x%02X). Reconnect in 3s...\n",
               p_data->connection_status.reason);
        conn_id = 0;
        target_char_handle = 0;
        vTaskDelay(pdMS_TO_TICKS(3000));
        wiced_bt_ble_scan(BTM_BLE_SCAN_TYPE_HIGH_DUTY, true, scan_result_cb);
    }
    break;
```

## Common BLE Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Calling `wiced_bt_stack_init()` before radio FW loaded | BLE does nothing, no events | Wait for `cy_wcm_init()` to complete first |
| Stack-local CCCD write buffer | Write "succeeds" but no notifications arrive | Make `cccd_val[]` and `write_hdr` static |
| Big-endian UUID comparison | Characteristic discovery never matches | Store UUID bytes in little-endian order |
| "First NOTIFY char" heuristic | Subscribes to wrong characteristic | Match by exact 128-bit UUID |
| No reconnect on disconnect | Device connects once, never recovers | Re-scan after disconnect with delay |
| Scanning during active connection | Wastes power, may cause instability | Stop scan before `gatt_le_connect()` |
| FreeRTOS task stack < 4096 for BLE | Stack overflow during GATT operations | Use 4096+ words for BLE task |
