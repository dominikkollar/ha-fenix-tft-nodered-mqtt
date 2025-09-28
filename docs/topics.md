## Schéma MQTT topiků 


### Přehled klíčových témat

* **Stav (publikace):**

  * `fenix/<MAC>/state/all` *(JSON: hvac_mode, current_temperature, target_temperature, floor_temperature)*
  * `fenix/<MAC>/state/current_temperature` *(°C)*
  * `fenix/<MAC>/state/target_temperature` *(°C; v OFF se nepotlačí do `state/all` → HA setpoint skryje)*
  * `fenix/<MAC>/state/floor_temperature` *(°C z `bo`, fallback `df`)*
  * `fenix/<MAC>/state/hvac_mode` *(off|heat|auto)*
  * `fenix/<MAC>/status` *(online/offline)*
  * Atributy (retained): `fenix/<MAC>/info/*`, `fenix/<MAC>/attributes`
* **Příkazy (odběr):**

  * `fenix/<MAC>/set` *(JSON: `{ "mode": "off|heat|auto|manual|schedule", "temperature": 22.5 }`)*
  * `fenix/<MAC>/set/mode` *(string)*
  * `fenix/<MAC>/set/temperature` *(number)*
* **ACK/Diagnostika:**

  * `fenix/<MAC>/ack/mode_ok`, `ack/control_ok`, `ack/setpoint_ok`, `ack/success`
  * `fenix/<MAC>/ack/error`
  * `fenix/<id>/diag/api_ok`, `diag/last_http_code`

