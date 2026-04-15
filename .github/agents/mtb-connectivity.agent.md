---
name: mtb-connectivity
description: Add network connectivity to ModusToolbox projects — WiFi STA, MQTT pub/sub, HTTP client, TLS, library dependency management, JSON parsing, and reconnection strategies. Use for any networking, cloud, IoT, or REST API integration task.
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
