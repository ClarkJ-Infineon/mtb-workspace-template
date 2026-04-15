---
name: ble-setup
description: Add Bluetooth Low Energy using BTSTACK v4 API to ModusToolbox projects with AIROC combo chips. Use when adding BLE, Bluetooth, GATT services, advertising, scanning, or BLE peripheral/central functionality.
---

# BLE Setup — BTSTACK v4 Patterns

Patterns for adding Bluetooth Low Energy (BLE) to ModusToolbox™ projects using the AIROC BTSTACK v4 API.

> **Applies to:** PSOC Edge, PSOC 6 with AIROC CYW43xxx/CYW55xxx combo chips.
> **Prerequisites:** Project must have been created via `project-creator-cli` (see /project-creation skill).

---

## Required Libraries

Add to `deps/` (or `proj_cm33_ns/deps/` for PSOC Edge):

```
https://github.com/Infineon/btstack-integration#latest-v5.X#$$ASSET_REPO$$/btstack-integration/latest-v5.X
https://github.com/Infineon/bluetooth-freertos#latest-v5.X#$$ASSET_REPO$$/bluetooth-freertos/latest-v5.X
https://github.com/Infineon/freertos#latest-v10.X#$$ASSET_REPO$$/freertos/latest-v10.X
https://github.com/cypresssemiconductorco/retarget-io#latest-v1.X#$$ASSET_REPO$$/retarget-io/latest-v1.X
```

## Required Makefile Configuration

```makefile
COMPONENTS+=FREERTOS BLUETOOTH
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF
```

> **Note:** Check the `btstack-integration` repo README for the current required COMPONENTS. BLE and BR/EDR may use different component names depending on the release.

---

## BLE Peripheral (GATT Server) Pattern

### Step 1: Define GATT Database

Create `cycfg_gatt_db.h` and `cycfg_gatt_db.c` using the Bluetooth Configurator in the MTB IDE, or define manually:

```c
/* app_bt_gatt_db.h — GATT attribute handles */
#define HDLS_CUSTOM_SERVICE              0x0028
#define HDLC_CUSTOM_SERVICE_CHAR         0x0029
#define HDLC_CUSTOM_SERVICE_CHAR_VALUE   0x002A
#define HDLD_CUSTOM_SERVICE_CHAR_CCC     0x002B  /* Client Config Descriptor */
```

### Step 2: BLE Stack Initialization

```c
#include "wiced_bt_stack.h"
#include "wiced_bt_dev.h"
#include "wiced_bt_ble.h"
#include "wiced_bt_gatt.h"
#include "cycfg_bt_settings.h"

/* BLE management callback — handles stack events */
wiced_result_t app_bt_management_callback(
    wiced_bt_management_evt_t event,
    wiced_bt_management_evt_data_t *p_event_data)
{
    switch (event)
    {
        case BTM_ENABLED_EVT:
            if (p_event_data->enabled.status == WICED_BT_SUCCESS)
            {
                printf("BLE stack initialized\n");

                /* Register GATT callback */
                wiced_bt_gatt_register(app_gatt_callback);

                /* Initialize GATT database */
                wiced_bt_gatt_db_init(gatt_database, gatt_database_len, NULL);

                /* Start BLE advertising */
                app_start_advertising();
            }
            break;

        case BTM_BLE_CONNECTION_PARAM_UPDATE:
            printf("BLE connection params updated\n");
            break;

        default:
            break;
    }
    return WICED_BT_SUCCESS;
}

void app_bt_init(void)
{
    /* Initialize BLE stack with configuration from Bluetooth Configurator */
    wiced_bt_stack_init(app_bt_management_callback, &wiced_bt_cfg_settings);
}
```

### Step 3: Advertising

```c
void app_start_advertising(void)
{
    wiced_bt_ble_advert_elem_t adv_elem[2];
    uint8_t num_elem = 0;

    /* Flags */
    uint8_t adv_flags = BTM_BLE_GENERAL_DISCOVERABLE_FLAG | BTM_BLE_BREDR_NOT_SUPPORTED;
    adv_elem[num_elem].advert_type = BTM_BLE_ADVERT_TYPE_FLAG;
    adv_elem[num_elem].len = 1;
    adv_elem[num_elem].p_data = &adv_flags;
    num_elem++;

    /* Complete Local Name */
    char device_name[] = "PSOC-BLE-Device";
    adv_elem[num_elem].advert_type = BTM_BLE_ADVERT_TYPE_NAME_COMPLETE;
    adv_elem[num_elem].len = strlen(device_name);
    adv_elem[num_elem].p_data = (uint8_t *)device_name;
    num_elem++;

    wiced_bt_ble_set_raw_advertisement_data(num_elem, adv_elem);
    wiced_bt_start_advertisements(
        BTM_BLE_ADVERT_UNDIRECTED_HIGH, BLE_ADDR_PUBLIC, NULL);
}
```

### Step 4: GATT Read/Write Handlers

```c
wiced_bt_gatt_status_t app_gatt_callback(
    wiced_bt_gatt_evt_t event,
    wiced_bt_gatt_event_data_t *p_event_data)
{
    switch (event)
    {
        case GATT_CONNECTION_STATUS_EVT:
            if (p_event_data->connection_status.connected)
                printf("BLE connected (conn_id=%d)\n",
                       p_event_data->connection_status.conn_id);
            else
                printf("BLE disconnected — restart advertising\n");
                app_start_advertising();
            break;

        case GATT_ATTRIBUTE_REQUEST_EVT:
            return app_gatt_attr_request_handler(
                &p_event_data->attribute_request);

        default:
            break;
    }
    return WICED_BT_GATT_SUCCESS;
}
```

---

## BLE Central (Scanner) Pattern

```c
void app_start_scan(void)
{
    wiced_bt_ble_scan(BTM_BLE_SCAN_TYPE_HIGH_DUTY, WICED_TRUE,
                      app_scan_result_callback);
}

void app_scan_result_callback(wiced_bt_ble_scan_results_t *p_scan_result,
                               uint8_t *p_adv_data)
{
    if (p_scan_result == NULL)
    {
        printf("Scan complete\n");
        return;
    }

    /* Parse advertisement data for device name */
    uint8_t *p_name = NULL;
    uint8_t name_len = 0;
    p_name = wiced_bt_ble_check_advertising_data(
        p_adv_data, BTM_BLE_ADVERT_TYPE_NAME_COMPLETE, &name_len);

    if (p_name != NULL)
    {
        printf("Found: %.*s [%02X:%02X:%02X:%02X:%02X:%02X] RSSI=%d\n",
               name_len, p_name,
               p_scan_result->remote_bd_addr[0],
               p_scan_result->remote_bd_addr[1],
               p_scan_result->remote_bd_addr[2],
               p_scan_result->remote_bd_addr[3],
               p_scan_result->remote_bd_addr[4],
               p_scan_result->remote_bd_addr[5],
               p_scan_result->rssi);
    }
}
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `BLUETOOTH` component | Linker errors on `wiced_bt_*` symbols | Add `COMPONENTS+=BLUETOOTH` to Makefile |
| Not calling `wiced_bt_gatt_register` before `gatt_db_init` | GATT callbacks never fire | Register callback in `BTM_ENABLED_EVT` handler |
| Not restarting advertising on disconnect | Device becomes invisible after first connection | Call `app_start_advertising()` in disconnect handler |
| Using deprecated BTSTACK v3 API | Compilation errors on newer BSPs | Use `wiced_bt_*` v4 API (check btstack-integration README) |
