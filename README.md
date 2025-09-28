# FENIX/WATTS TFT ↔ Home Assistant (Node-RED + MQTT)

A Node-RED flow that integrates the **FENIX / WATTS TFT Wi-Fi** thermostat with Home Assistant over MQTT.
It can:

* periodically read states from the REST API (At/Sp/Ma/Dm/Cm/bo/df…),
* publish states to MQTT (including aggregated `state/all`),
* receive commands from MQTT (`mode`, `temperature`) and apply them to the API,
* verify changes via subsequent `GET` calls (Dm/Cm/Ma/Sp),
* refresh the OAuth2 token and retry HTTP with exponential backoff,
* perform automatic **MQTT Discovery** for the `climate` domain in Home Assistant,
* provide an “optimistic UI” — immediately switch state in HA after a successful `PUT`.

> ⚠️ The flow works with API temperatures in **Fahrenheit × 10** (e.g., `752 = 75.2 °F`). It publishes **°C** to MQTT with 0.1 precision.

---

## Contents

* [Architecture](#architecture)
* [Requirements](#requirements)
* [Installation](#installation)
* [Configuration](#configuration)
* [MQTT Topics](#mqtt-topics)
* [Home Assistant Discovery](#home-assistant-discovery)
* [HTTP Calls & Verification](#http-calls--verification)
* [Value Mapping](#value-mapping)
* [Diagnostics & Logs](#diagnostics--logs)
* [Troubleshooting](#troubleshooting)
* [Security](#security)

---

## Architecture

* **Polling:** every 60 s `GET /businessmodule/v1/installations/admins/{installationId}` → list of rooms/devices → for each device `GET /iotmanagement/v1/configuration/{id}/{id}/v1/content`.
* **Parser:** convert F×10 → °C, determine mode via `Dm`/`Cm`, floor temperature from `bo` (fallback `df`), build `state/all`.
* **Commands:** MQTT commands → `PUT /iotmanagement/v1/devices/twin/properties/config/replace` with `Dm`/`Cm`/`Ma`/`Sp` items.
* **Verify:** after ~1.2 s `GET …/content/{Dm|Cm|Ma|Sp}` → ACK topics + final `state/all`.
* **Auth:** periodic token refresh (55 min) + “ensure refresh” before poll/command.
* **HTTP retry:** subflow with exponential backoff and a whitelist of codes (429/5xx).

---

## Requirements

* Node-RED ≥ 3.x, nodes **http request**, **mqtt in/out**.
* An MQTT broker accessible from both Node-RED and Home Assistant.
* Access to the FENIX/WATTS API: `client_id`, `client_secret`, `refresh_token` (optionally `basic_auth` base64), and optionally `ocp-apim-subscription-key`.

---

## Installation

1. In Node-RED: **Menu → Import** and paste the flow JSON.
2. Open the MQTT broker configs (`MQTT (LWT set here)` for publishing, `MQTT (edit me)` for subscribing).
3. Store credentials in the **flow context** (see below).
4. Set `fenix_installation_id` (e.g., `76103DB2785A`).
5. Click the **Set refresh_token…** inject (or wait for automatic refresh).
6. After the first poll, climate entities will appear in HA via MQTT Discovery.

---

## Configuration

The flow reads keys from `flow.http_cons`:

```json
{
  "client_id": "…",
  "client_secret": "…",
  "refresh_token": "…",
  "access_token": "… (optional, will be refreshed)",
  "token_type": "Bearer",
  "expires_at": 0,                       // epoch in s/ms (flow will normalize)
  "subscription_key": "… (if APIM requires it)",
  "basic_auth": "base64(client_id:client_secret) (optional)"
}
```

Other keys in flow context:

* `fenix_installation_id`: installation ID (for device discovery).
* `fenix_device_ids`: array of MAC/ID (optional; otherwise loaded from the installation).
* `ha_discovery_prefix`: HA discovery prefix (default `homeassistant`).

---

## MQTT Topics

### Published (Node-RED → MQTT)

* Availability: `fenix/<MAC>/status` → `online|offline` (retained; configure HA LWT in the broker node).
* State:

  * `fenix/<MAC>/state/current_temperature` (°C)
  * `fenix/<MAC>/state/target_temperature` (°C, **not published** when `hvac_mode='off'`)
  * `fenix/<MAC>/state/floor_temperature` (°C, from `bo`, fallback `df`)
  * `fenix/<MAC>/state/hvac_mode` (`off|heat|auto`)
  * **aggregated** `fenix/<MAC>/state/all` (JSON):

    ```json
    {
      "hvac_mode": "off|heat|auto",
      "current_temperature": 21.3,
      "target_temperature": 22.0,   // null in OFF → HA hides the setpoint
      "floor_temperature": 20.1
    }
    ```
* Attributes (retained):

  * `fenix/<MAC>/info/name`, `/info/version`, `/info/tz`,
  * `fenix/<MAC>/info/floor_limit_c` (from `df`),
  * `fenix/<MAC>/attributes` (JSON: identification, version, last update, TZ…).
* ACK/diag:

  * `fenix/<MAC>/ack/mode_ok`, `…/ack/control_ok`, `…/ack/setpoint_ok`, `…/ack/success` → `true|false`
  * `fenix/<MAC>/ack/error` → error text (e.g., `PUT 401`)
  * `fenix/<id>/diag/api_ok`, `…/diag/last_http_code`

### Subscribed (HA → Node-RED)

* `fenix/<MAC>/set` (JSON):

  ```json
  { "mode": "off|heat|auto|manual|schedule", "temperature": 22.5 }
  ```
* `fenix/<MAC>/set/mode` (`off|heat|auto|manual|schedule`)
* `fenix/<MAC>/set/temperature` (number, °C)

> Note: `heat` is mapped internally to **manual**, `auto` to **schedule**.

---

## Home Assistant Discovery

The flow publishes a retained `config` for `climate`:

* Topic: `<prefix>/climate/fenix_tft_<MAC>/config` (default prefix `homeassistant`)
* Bindings:

  * `mode_state_topic` / `current_temperature_topic` / `temperature_state_topic` → `fenix/<MAC>/state/all` + Jinja templates.
  * `mode_command_topic`: `fenix/<MAC>/set/mode`
  * `temperature_command_topic`: `fenix/<MAC>/set/temperature`
* `modes: ["off","heat","auto"]`, `temp_step: 0.5`, `precision: 0.1`, `min_temp: 5`, `max_temp: 35`.

No YAML config needed — only an active MQTT integration in HA.

---

## HTTP Calls & Verification

### Changing mode/setpoint (PUT)

`PUT https://vs2-fe-apim-prod.azure-api.net/iotmanagement/v1/devices/twin/properties/config/replace`

Body (example **manual + 22.0 °C**):

```json
{
  "Id_deviceId": "AC67B2DD98F0",
  "S1": "AC67B2DD98F0",
  "configurationVersion": "v1.0",
  "data": [
    { "wattsType": "Dm", "wattsTypeValue": 6 },
    { "wattsType": "Ma", "wattsTypeValue": 719 } // 22.0 °C → 71.6 °F → 716 (rounded to 5 → 719/715 depending on FW)
  ]
}
```

Modes:

* **manual**: `Dm=6` (+ `Ma` if temperature is included)
* **off**: `Dm=0`
* **schedule/auto**: `Cm=1` (+ `Sp` if temperature is included) and defensively `Dm=1` (not off)

> After a successful `PUT`, the flow publishes **optimistic** `state/all` so HA switches immediately. The follow-up verification confirms/adjusts it.

### Verification (GET)

After ~1.2 s:

* `GET …/v1/content/Dm`, `…/Cm`, `…/Ma` or `…/Sp`
  Compares the `value` against the expected one; publishes `ack/*` and may update `state/all`.

---

## Value Mapping

* **Temperatures:** API `F×10` → MQTT **°C** (0.1).
* **current:** `At`
* **target:** `Sp` (schedule) **or** `Ma` (manual)
* **floor:** `bo` (if missing, `df` — also the minimum floor limit → `info/floor_limit_c`)
* **mode:**

  * `Dm=0` → `off`
  * `Cm=1` → `auto`
  * otherwise → `heat` (manual)

---

## Diagnostics & Logs

* HTTP retry subflow (`429,500,502,503,504` + network errors), exponential backoff (base 1000 ms, max 30 s).
* MQTT diagnostics:

  * `fenix/<id>/diag/api_ok`, `…/diag/last_http_code`
  * `fenix/<id>/ack/error` (error text)
* The **MQTT publish (array)** node validates topics (blocks `+`, `#`, and whitespace).

---

## Troubleshooting

* **HA doesn’t switch immediately after a command**
  The flow publishes an “optimistic” `state/all` right after a `PUT` 2xx. If you don’t see it, check:

  * that the climate entity subscribes to `state/all` (discovery),
  * that the HA MQTT client sees `fenix/<MAC>/state/all` and `fenix/<MAC>/state/hvac_mode`.

* **“TypeError: Cannot read properties of undefined (reading 'expect')”**
  Addressed: the verification node now guards missing `msg.item/expect` and only warns into the log. If it appears again, verify that **Build GET verifications** actually created at least one item (typically when you send only a mode without a temperature).

* **Floor temperature looks wrong**
  `bo` is used (fallback `df`). `df` is additionally published to `info/floor_limit_c`.

* **PUT 401/403**
  Check `flow.http_cons` (valid `refresh_token`, `client_id/secret`, optionally `subscription_key`). Keep debug nodes that print sensitive data disabled.

---

## Security

* Keep tokens and keys in the **flow context**.
* Leave debugs that would display the `access_token` disabled by default.
* If APIM doesn’t require `ocp-apim-subscription-key`, don’t add the header.

---

## Quick Checklist

1. Fill in `flow.http_cons` and `fenix_installation_id`.
2. Configure MQTT brokers.
3. Trigger token refresh (inject) or wait for auto-refresh.
4. Check that **climate** entities appeared in HA and `state/all` is updating.
5. Test commands:

   * `fenix/<MAC>/set/mode` → `off|heat|auto`
   * `fenix/<MAC>/set/temperature` → e.g., `22.5`

---
