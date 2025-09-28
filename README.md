# FENIX/WATTS TFT ↔ Home Assistant (Node-RED + MQTT)

Node-RED flow, který integruje termostat **FENIX / WATTS TFT Wi-Fi** do Home Assistantu přes MQTT.
Umí:

* pravidelné čtení stavů z REST API (At/Sp/Ma/Dm/Cm/bo/df…),
* publikaci stavů do MQTT (včetně sdruženého `state/all`),
* příjem příkazů z MQTT (`mode`, `temperature`) a jejich promítnutí do API,
* verifikaci změn přes následné `GET` (Dm/Cm/Ma/Sp),
* obnovu OAuth2 tokenu a HTTP retry s exponenciálním backoffem,
* automatický **MQTT Discovery** pro `climate` v Home Assistantu,
* “optimistic UI” — okamžité přepnutí stavu v HA po úspěšném `PUT`.

> ⚠️ Flow pracuje s teplotami API ve **Fahrenheit × 10** (např. `752 = 75.2 °F`). V MQTT publikuje **°C** s přesností 0.1.

---

## Obsah

* [Architektura](#architektura)
* [Požadavky](#požadavky)
* [Instalace](#instalace)
* [Konfigurace](#konfigurace)
* [MQTT topiky](#mqtt-topiky)
* [Home Assistant Discovery](#home-assistant-discovery)
* [HTTP volání a ověřování](#http-volání-a-ověřování)
* [Mapování hodnot](#mapování-hodnot)
* [Diagnostika a logy](#diagnostika-a-logy)
* [Troubleshooting](#troubleshooting)
* [Bezpečnost](#bezpečnost)

---

## Architektura

* **Polling**: každých 60 s `GET /businessmodule/v1/installations/admins/{installationId}` → seznam místností/zařízení → pro každé zařízení `GET /iotmanagement/v1/configuration/{id}/{id}/v1/content`.
* **Parser**: převod F×10 → °C, zjištění režimu podle `Dm`/`Cm`, floor teploty z `bo` (fallback `df`), tvorba `state/all`.
* **Commands**: MQTT příkazy → `PUT /iotmanagement/v1/devices/twin/properties/config/replace` s položkami `Dm`/`Cm`/`Ma`/`Sp`.
* **Verify**: po ~1.2 s `GET …/content/{Dm|Cm|Ma|Sp}` → ACK topiky + finální `state/all`.
* **Auth**: periodický refresh tokenu (55 min) + “ensure refresh” před poll/command.
* **HTTP retry**: subflow s exponenciálním backoffem a whitelistem kódů (429/5xx).

---

## Požadavky

* Node-RED ≥ 3.x, uzel **http request**, **mqtt in/out**.
* MQTT broker dostupný z Node-RED i Home Assistantu.
* Přístup do FENIX/WATTS API: `client_id`, `client_secret`, `refresh_token` (příp. `basic_auth` base64), volitelně `ocp-apim-subscription-key`.

---

## Instalace

1. V Node-RED: **Menu → Import** a vlož JSON flow.
2. Otevři konfiguraci MQTT brokerů (`MQTT (LWT set here)` pro publikaci, `MQTT (edit me)` pro příjem).
3. Do **flow contextu** ulož přístupové údaje (viz níže).
4. Nastraž `fenix_installation_id` (např. `76103DB2785A`).
5. Stiskni inject **Set refresh_token…** (nebo počkej na automatický refresh).
6. Po prvním pollu se v HA objeví climate entity přes MQTT Discovery.

---

## Konfigurace

Flow čte klíče z `flow.http_cons`:

```json
{
  "client_id": "…",
  "client_secret": "…",
  "refresh_token": "…",
  "access_token": "… (volitelné, načte se refresh)",
  "token_type": "Bearer",
  "expires_at": 0,                       // epoch v s/ms (flow si dopočítá)
  "subscription_key": "… (pokud APIM vyžaduje)",
  "basic_auth": "base64(client_id:client_secret) (volitelné)"
}
```

Další klíče ve flow contextu:

* `fenix_installation_id`: ID instalace (pro discovery zařízení).
* `fenix_device_ids`: pole MAC/ID (volitelné; jinak se načte z instalace).
* `ha_discovery_prefix`: prefix pro HA discovery (default `homeassistant`).

---

## MQTT topiky

### Publikované (Node-RED → MQTT)

* Dostupnost: `fenix/<MAC>/status` → `online|offline` (retained pro HA LWT si nastav v broker uzlu).
* Stavové:

  * `fenix/<MAC>/state/current_temperature` (°C)
  * `fenix/<MAC>/state/target_temperature` (°C, **nepublikuje se** pokud `hvac_mode='off'`)
  * `fenix/<MAC>/state/floor_temperature` (°C, z `bo`, fallback `df`)
  * `fenix/<MAC>/state/hvac_mode` (`off|heat|auto`)
  * **sdružené** `fenix/<MAC>/state/all` (JSON):

    ```json
    {
      "hvac_mode": "off|heat|auto",
      "current_temperature": 21.3,
      "target_temperature": 22.0,   // v OFF je null → HA skryje setpoint
      "floor_temperature": 20.1
    }
    ```
* Atributy (retained):

  * `fenix/<MAC>/info/name`, `/info/version`, `/info/tz`,
  * `fenix/<MAC>/info/floor_limit_c` (z `df`),
  * `fenix/<MAC>/attributes` (JSON: identifikace, verze, poslední update, TZ…).
* ACK/diag:

  * `fenix/<MAC>/ack/mode_ok`, `…/ack/control_ok`, `…/ack/setpoint_ok`, `…/ack/success` → `true|false`
  * `fenix/<MAC>/ack/error` → text chyby (např. `PUT 401`)
  * `fenix/<id>/diag/api_ok`, `…/diag/last_http_code`

### Odběry (HA → Node-RED)

* `fenix/<MAC>/set` (JSON):

  ```json
  { "mode": "off|heat|auto|manual|schedule", "temperature": 22.5 }
  ```
* `fenix/<MAC>/set/mode` (`off|heat|auto|manual|schedule`)
* `fenix/<MAC>/set/temperature` (číslo °C)

> Pozn.: `heat` se interně mapuje na **manual**, `auto` na **schedule**.

---

## Home Assistant Discovery

Flow publikuje retained `config` pro `climate`:

* Topic: `<prefix>/climate/fenix_tft_<MAC>/config` (default prefix `homeassistant`)
* Přiřazení:

  * `mode_state_topic` / `current_temperature_topic` / `temperature_state_topic` → `fenix/<MAC>/state/all` + Jinja šablony.
  * `mode_command_topic`: `fenix/<MAC>/set/mode`
  * `temperature_command_topic`: `fenix/<MAC>/set/temperature`
* `modes: ["off","heat","auto"]`, `temp_step: 0.5`, `precision: 0.1`, `min_temp: 5`, `max_temp: 35`.

Není potřeba žádná YAML konfigurace — pouze aktivní MQTT integrace v HA.

---

## HTTP volání a ověřování

### Změna režimu/setpointu (PUT)

`PUT https://vs2-fe-apim-prod.azure-api.net/iotmanagement/v1/devices/twin/properties/config/replace`

Tělo (příklad **manual + 22.0 °C**):

```json
{
  "Id_deviceId": "AC67B2DD98F0",
  "S1": "AC67B2DD98F0",
  "configurationVersion": "v1.0",
  "data": [
    { "wattsType": "Dm", "wattsTypeValue": 6 },
    { "wattsType": "Ma", "wattsTypeValue": 719 } // 22.0 °C → 71.6 °F → 716 (zaokrouhleno na 5 → 719/715 dle FW)
  ]
}
```

Režimy:

* **manual**: `Dm=6` (+ `Ma` pokud je teplota)
* **off**: `Dm=0`
* **schedule/auto**: `Cm=1` (+ `Sp` pokud je teplota) a ochranně `Dm=1` (není off)

> Flow po úspěšném `PUT` publikuje **optimistic** `state/all`, aby se HA přepnul okamžitě. Následná verifikace to potvrdí/doopraví.

### Verifikace (GET)

Po ~1.2 s:

* `GET …/v1/content/Dm`, `…/Cm`, `…/Ma` nebo `…/Sp`
  Porovná se `value` s očekávanou hodnotou; publikuje se `ack/*` a případně se aktualizuje `state/all`.

---

## Mapování hodnot

* **Teploty**: API `F×10` → MQTT **°C** (0.1).
* **current**: `At`
* **target**: `Sp` (schedule) **nebo** `Ma` (manual)
* **floor**: `bo` (pokud není, `df` – zároveň minimální limit podlahy → `info/floor_limit_c`)
* **režim**:

  * `Dm=0` → `off`
  * `Cm=1` → `auto`
  * jinak → `heat` (manual)

---

## Diagnostika a logy

* HTTP retry subflow (`429,500,502,503,504` + síťové chyby), exponenciální backoff (base 1000 ms, max 30 s).
* MQTT diagnostika:

  * `fenix/<id>/diag/api_ok`, `…/diag/last_http_code`
  * `fenix/<id>/ack/error` (text chyby)
* Uzel **MQTT publish (array)** validuje topiky (blokuje `+`, `#` a whitespace).

---

## Troubleshooting

* **HA se hned nepřepíná po příkazu**
  Flow publikuje “optimistic” `state/all` hned po 2xx z `PUT`. Pokud to nevidíš, zkontroluj:

  * že climate entity odebírá `state/all` (discovery),
  * že MQTT klient HA vidí topiky `fenix/<MAC>/state/all` a `fenix/<MAC>/state/hvac_mode`.

* **„TypeError: Cannot read properties of undefined (reading 'expect')“**
  Ošetřeno: verifikační uzel nyní guarduje chybějící `msg.item/expect` a jen varuje do logu. Pokud by se znovu objevilo, zkontroluj, zda `Build GET verifications` skutečně vytvořil alespoň jednu položku (typicky když posíláš jen mód bez teploty).

* **Floor teplota je divná**
  Používá se `bo` (back-up `df`). `df` se navíc publikuje do `info/floor_limit_c`.

* **PUT 401/403**
  Zkontroluj `flow.http_cons` (platný `refresh_token`, `client_id/secret`, případně `subscription_key`). V debug uzlech vypínej citlivá data.

---

## Bezpečnost

* Tokeny a klíče drž ve **flow contextu**.
* Debugy, které by zobrazovaly `access_token`, nech výchozím stavem vypnuté.
* Pokud APIM nevyžaduje `ocp-apim-subscription-key`, header nepřidávej.

---

## Rychlý checklist

1. Vyplň `flow.http_cons` a `fenix_installation_id`.
2. Nastav MQTT brokery.
3. Spusť refresh tokenu (inject) nebo počkej na auto-refresh.
4. Zkontroluj, že se v HA objevily **climate** entity a mění se `state/all`.
5. Otestuj příkazy:

   * `fenix/<MAC>/set/mode` → `off|heat|auto`
   * `fenix/<MAC>/set/temperature` → např. `22.5`

---

