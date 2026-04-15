---
name: wifi-mqtt
description: Add WiFi STA connectivity and MQTT publish/subscribe to ModusToolbox projects. Use when adding WiFi, MQTT, network connectivity, cloud communication, or IoT telemetry publishing to a project.
---

# WiFi + MQTT — Connection, Publish, TLS

Patterns for adding WiFi STA connectivity and MQTT publish/subscribe to ModusToolbox™ projects.

> **Applies to:** PSOC Edge (CM33 NS), PSOC 6, and any kit with CYW43xxx/CYW55xxx WiFi radio.
> **Prerequisites:** Project must have been created via `project-creator-cli` (see /project-creation skill).

---

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

Run `make getlibs` after adding.

## Required Makefile Configuration

```makefile
COMPONENTS+=FREERTOS LWIP MBEDTLS
DEFINES+=CYBSP_WIFI_CAPABLE
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF
```

---

## WiFi STA Connection Pattern

```c
#include "cy_wcm.h"

/* WiFi credentials — replace with your values or use provisioning */
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
                   (ip_addr.ip.v4 & 0xFF),
                   (ip_addr.ip.v4 >> 8) & 0xFF,
                   (ip_addr.ip.v4 >> 16) & 0xFF,
                   (ip_addr.ip.v4 >> 24) & 0xFF);
            return CY_RSLT_SUCCESS;
        }
        printf("WiFi connect failed (attempt %d/%d), retrying...\n",
               retry + 1, MAX_WIFI_RETRIES);
        vTaskDelay(pdMS_TO_TICKS(WIFI_RETRY_DELAY_MS));
    }
    return result;
}
```

---

## MQTT Connection + Publish Pattern

```c
#include "cy_mqtt_api.h"

#define MQTT_BROKER_URL     "mqtt.example.com"
#define MQTT_BROKER_PORT    8883        /* TLS */
#define MQTT_CLIENT_ID      "psoc-edge-device-01"
#define MQTT_TOPIC          "devices/psoc-edge/telemetry"
#define MQTT_QOS            CY_MQTT_QOS1

static cy_mqtt_t mqtt_handle;

/* TLS root CA certificate — PEM format, null-terminated */
static const char root_ca_cert[] =
    "-----BEGIN CERTIFICATE-----\n"
    "... your broker's root CA here ...\n"
    "-----END CERTIFICATE-----\n";

cy_rslt_t mqtt_connect(void)
{
    /* Initialize MQTT library (call once) */
    cy_rslt_t result = cy_mqtt_init();
    if (result != CY_RSLT_SUCCESS) return result;

    /* Broker info with TLS */
    cy_mqtt_broker_info_t broker_info = {
        .hostname      = MQTT_BROKER_URL,
        .hostname_len  = strlen(MQTT_BROKER_URL),
        .port          = MQTT_BROKER_PORT,
    };

    /* TLS credentials */
    cy_awsport_ssl_credentials_t tls_creds = {
        .root_ca         = root_ca_cert,
        .root_ca_size    = sizeof(root_ca_cert),
    };

    /* Create MQTT instance */
    result = cy_mqtt_create(NULL, NULL, NULL,
                            &broker_info, &tls_creds,
                            &mqtt_handle);
    if (result != CY_RSLT_SUCCESS) return result;

    /* Connect */
    cy_mqtt_connect_info_t conn_info = {
        .client_id      = MQTT_CLIENT_ID,
        .client_id_len  = strlen(MQTT_CLIENT_ID),
        .keep_alive_sec = 60,
        .clean_session  = true,
    };

    result = cy_mqtt_connect(mqtt_handle, &conn_info);
    return result;
}

cy_rslt_t mqtt_publish(const char *topic, const char *payload)
{
    cy_mqtt_publish_info_t pub_info = {
        .qos        = MQTT_QOS,
        .topic      = topic,
        .topic_len  = strlen(topic),
        .payload    = (const char *)payload,
        .payload_len = strlen(payload),
    };

    return cy_mqtt_publish(mqtt_handle, &pub_info);
}
```

---

## MQTT Subscribe + Callback Pattern

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
        .qos       = MQTT_QOS,
        .topic     = topic,
        .topic_len = strlen(topic),
    };

    return cy_mqtt_subscribe(mqtt_handle, &sub_info, 1);
}
```

---

## Reconnection Strategy

WiFi and MQTT connections drop in real deployments. Use a reconnection task:

```c
void network_monitor_task(void *arg)
{
    for (;;)
    {
        vTaskDelay(pdMS_TO_TICKS(10000));  /* Check every 10s */

        if (!cy_wcm_is_connected_to_ap())
        {
            printf("WiFi lost — reconnecting...\n");
            wifi_connect();
            mqtt_connect();  /* Must reconnect MQTT after WiFi recovery */
        }
    }
}
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `CYBSP_WIFI_CAPABLE` define | WiFi init fails silently | Add to Makefile DEFINES |
| Missing `LWIP` or `MBEDTLS` component | Linker errors on cy_wcm/cy_mqtt symbols | Add to Makefile COMPONENTS |
| Non-TLS port (1883) in production | Data sent in plaintext | Use port 8883 with root CA |
| No reconnection logic | Device goes offline permanently after WiFi blip | Add network monitor task |
| Publishing faster than TLS can encrypt | MQTT queue backlog, eventual disconnect | Rate-limit publishes (1-2/sec typical) |
