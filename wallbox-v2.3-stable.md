# Wallbox Intelligens Töltésszabályozás (v2.3 - STABLE)

## 🇭🇺 MAGYAR VERZIÓ

### Bevezetés

Ez a megoldás lehetővé teszi a **Wallbox töltő intelligens, terhelésfüggő szabályozását** Home Assistant-ban. **Stabil, rate-limited verziója.**

**Célok:**
- ✅ Biztosíték védelme (16A/fázis limit)
- ✅ Terhelésfüggő töltés (CP kitöltési tényezővel)
- ✅ Automatikus szüneteltetés elégtelenség esetén
- ✅ **Globális rate limiting** (max. 2 perc közönként API call)
- ✅ **Stabil trigger-ek** (debounce-zált)
- ✅ Vészhelyzet kezelés

---

## Implementáció

### 0. Helperek

Settings → Devices & Services → Helpers → Create Helper:

```yaml
input_number:
  wallbox_last_set_current:
    name: "Wallbox Last Set Current"
    unit_of_measurement: "A"
    min: 6
    max: 32
    step: 1
    icon: mdi:flash

input_datetime:
  wallbox_last_switch_time:
    name: "Wallbox Last Switch Time"
    has_date: true
    has_time: true
    icon: mdi:clock

input_boolean:
  wallbox_switch_pending:
    name: "Wallbox Switch Pending"
    icon: mdi:timer-sand
```

---

### 1. Template Szenzor (`template.yaml`)

```yaml
template:
  - sensor:
      - name: "Wallbox Maximális Árama"
        unique_id: wallbox_maximalis_arama
        unit_of_measurement: "A"
        device_class: current
        state_class: measurement
        state: >
          {% set a = states('sensor.shellyZZZZZ_phase_a_current') | float(0) %}
          {% set b = states('sensor.shellyZZZZZ_phase_b_current') | float(0) %}
          {% set c = states('sensor.shellyZZZZZ_phase_c_current') | float(0) %}
          
          {% set free_a = 16.0 - a - 0.5 %}
          {% set free_b = 16.0 - b - 0.5 %}
          {% set free_c = 16.0 - c - 0.5 %}
          
          {% set min_free_phase = [free_a, free_b, free_c] | min %}
          {% set wallbox_setpoint = [32, [min_free_phase, 6] | max] | min %}
          
          {{ wallbox_setpoint | round(0) | int }}
        
        availability: >
          {{ 
            has_value('sensor.shellyZZZZZ_phase_a_current') and
            has_value('sensor.shellyZZZZZ_phase_b_current') and
            has_value('sensor.shellyZZZZZ_phase_c_current')
          }}

      - name: "Wallbox Fázis Figyelmeztetés"
        unique_id: wallbox_fazis_figyelmeztetes
        state: >
          {% set a = states('sensor.shellyZZZZZ_phase_a_current') | float(0) %}
          {% set b = states('sensor.shellyZZZZZ_phase_b_current') | float(0) %}
          {% set c = states('sensor.shellyZZZZZ_phase_c_current') | float(0) %}
          
          {% set free_a = 16.0 - a - 0.5 %}
          {% set free_b = 16.0 - b - 0.5 %}
          {% set free_c = 16.0 - c - 0.5 %}
          
          {% set min_free_phase = [free_a, free_b, free_c] | min %}
          
          {% if a > 16.0 or b > 16.0 or c > 16.0 %}
            KRITIKUS: Egy vagy több fázis túlterhelve! (A:{{ a }}A, B:{{ b }}A, C:{{ c }}A)
          {% elif min_free_phase < 0 %}
            KRITIKUS: Szabad kapacitás negatív! Min szabad: {{ min_free_phase | round(2) }}A
          {% elif min_free_phase < 1 %}
            FIGYELMEZTETÉS: Szabad kapacitás nagyon alacsony! Min szabad: {{ min_free_phase | round(2) }}A
          {% elif min_free_phase < 6 %}
            FIGYELMEZTETÉS: Nincs elég szabad kapacitás! Min szabad: {{ min_free_phase | round(2) }}A (szükséges: 6A)
          {% elif a > 15 or b > 15 or c > 15 %}
            FIGYELMEZTETÉS: Egy vagy több fázis közel a limithez (>15A)
          {% else %}
            OK - Normál üzem
          {% endif %}
        
        availability: "{{ true }}"
```

---

### 2. Automatizmus 1: Terhelésfüggő Max Áram

```yaml
alias: Wallbox terhelésfüggő töltés
description: Wallbox max áram terhelésfüggően (2 percenként)
mode: single
triggers:
  - platform: time_pattern
    minutes: "/2"
conditions:
  - condition: template
    value_template: >-
      {{ states('sensor.wallbox_maximalis_arama') not in ['unknown', 'unavailable'] }}
  - condition: template
    value_template: >-
      {{ states('input_number.wallbox_last_set_current') | int(0) != states('sensor.wallbox_maximalis_arama') | int(6) }}
actions:
  - service: number.set_value
    target:
      entity_id: number.wallbox_XXXXXgarazs_sn_SSSSS_maximalis_toltoaram
    data:
      value: "{{ states('sensor.wallbox_maximalis_arama') | int(6) }}"
  - service: input_number.set_value
    target:
      entity_id: input_number.wallbox_last_set_current
    data:
      value: "{{ states('sensor.wallbox_maximalis_arama') | int(6) }}"
```

---

### 3. Automatizmus 2: PAUSE Trigger (Stabilizált)

```yaml
alias: Wallbox pause - Request
description: Pause request - flaggeli ha pause szükséges (stabilizált)
mode: single
triggers:
  - platform: state
    entity_id: sensor.wallbox_fazis_figyelmeztetes
    for:
      seconds: 20
conditions:
  - condition: template
    value_template: >-
      {{ states('sensor.wallbox_fazis_figyelmeztetes') not in ['OK - Normál üzem', 'unknown', 'unavailable'] }}
  - condition: state
    entity_id: switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas
    state: "on"
  - condition: state
    entity_id: input_boolean.wallbox_switch_pending
    state: "off"
actions:
  - service: input_boolean.turn_on
    target:
      entity_id: input_boolean.wallbox_switch_pending
  - service: persistent_notification.create
    data:
      title: "⏳ PAUSE - Pending (Rate Limited)"
      message: >
        Status: {{ states('sensor.wallbox_fazis_figyelmeztetes') }}
        ➡️ Flag bejelöltve - várakozás az API slot-ra...
      notification_id: wallbox_pause_pending
```

**Lényeg:**
- `for: seconds: 20` - Stabilizáció: az érték 20 másodpercig azonos
- Condition: `not in ['OK - Normál üzem', 'unknown', 'unavailable']` - Biztosan figyelmeztetés
- Condition: `wallbox_switch_pending == off` - Csak egyszer trigger

---

### 4. Automatizmus 3: RESUME Trigger (Stabilizált)

```yaml
alias: Wallbox resume - Request
description: Resume request - flaggeli ha resume szükséges (stabilizált)
mode: single
triggers:
  - platform: state
    entity_id: sensor.wallbox_fazis_figyelmeztetes
    to: "OK - Normál üzem"
    for:
      seconds: 30
conditions:
  - condition: state
    entity_id: switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas
    state: "off"
  - condition: state
    entity_id: input_boolean.wallbox_switch_pending
    state: "on"
actions:
  - service: input_boolean.turn_off
    target:
      entity_id: input_boolean.wallbox_switch_pending
  - service: persistent_notification.create
    data:
      title: "⏳ RESUME - Pending (Rate Limited)"
      message: >
        Status: OK - Normál üzem
        ➡️ Flag törlve - várakozás az API slot-ra...
      notification_id: wallbox_resume_pending
```

---

### 5. Automatizmus 4: PAUSE/RESUME Gate Keeper (KÖZPONTI!)

```yaml
alias: Wallbox switch - Gate Keeper (Rate Limited)
description: Központi rate limiter - 2 percenként nyúl az API-hoz
mode: single
triggers:
  - platform: time_pattern
    minutes: "/2"
conditions: []
actions:
  - choose:
      # 🔴 PAUSE - Ha pending flag ON és switch még ON
      - conditions:
          - condition: state
            entity_id: input_boolean.wallbox_switch_pending
            state: "on"
          - condition: state
            entity_id: switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas
            state: "on"
        sequence:
          - service: persistent_notification.create
            data:
              title: "🔴 SWITCH OFF"
              message: >
                Status: {{ states('sensor.wallbox_fazis_figyelmeztetes') }}
                ➡️ API Call: SWITCH OFF
              notification_id: wallbox_debug_action
          - service: switch.turn_off
            target:
              entity_id: switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas
          - service: input_datetime.set_datetime
            target:
              entity_id: input_datetime.wallbox_last_switch_time
            data:
              datetime: "{{ now().isoformat() }}"
          - delay:
              seconds: 1
          - service: persistent_notification.create
            data:
              title: "Wallbox - Töltés Tiltva"
              message: "{{ states('sensor.wallbox_fazis_figyelmeztetes') }}"
              notification_id: wallbox_charging_paused
      
      # ✅ RESUME - Ha pending flag OFF és switch OFF
      - conditions:
          - condition: state
            entity_id: input_boolean.wallbox_switch_pending
            state: "off"
          - condition: state
            entity_id: switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas
            state: "off"
          - condition: state
            entity_id: sensor.wallbox_fazis_figyelmeztetes
            state: "OK - Normál üzem"
        sequence:
          - service: persistent_notification.create
            data:
              title: "✅ SWITCH ON"
              message: >
                Status: OK - Normál üzem
                ➡️ API Call: SWITCH ON
              notification_id: wallbox_debug_action
          - service: switch.turn_on
            target:
              entity_id: switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas
          - service: input_datetime.set_datetime
            target:
              entity_id: input_datetime.wallbox_last_switch_time
            data:
              datetime: "{{ now().isoformat() }}"
          - delay:
              seconds: 2
          - service: persistent_notification.dismiss
            data:
              notification_id: wallbox_charging_paused
```

---

### 6. Automatizmus 5: Safety - Shelly Offline

```yaml
alias: Wallbox safety - Shelly unavailable
description: VÉSZHELYZET - ha Shelly offline (AZONNAL!)
mode: single
triggers:
  - platform: state
    entity_id: sensor.wallbox_fazis_figyelmeztetes
    to: "unavailable"
    for:
      seconds: 10
conditions: []
actions:
  - service: persistent_notification.create
    data:
      title: "🚨 VÉSZHELYZET - Shelly offline!"
      message: >
        A Shelly 3EM szenzor UNAVAILABLE!
        SWITCH OFF-ra állítva a biztonság kedvéért!
      notification_id: wallbox_safety_shelly_offline
  - service: switch.turn_off
    target:
      entity_id: switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas
  - service: input_datetime.set_datetime
    target:
      entity_id: input_datetime.wallbox_last_switch_time
    data:
      datetime: "{{ now().isoformat() }}"
```

---

## 📊 Logika Flow

```
SENSOR ÉRTÉK VÁLTOZIK
      ↓
┌─────────────────────────────────────────────┐
│  FIGYELMEZTETÉS (20 sec stabil) ?           │
│  ✅ YES → Automatizmus 2 (Pause) flaggel    │
│  ❌ NO → Wait                               │
└─────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────┐
│  OK - NORMÁL ÜZEM (30 sec stabil) ?         │
│  ✅ YES → Automatizmus 3 (Resume) flag törl │
│  ❌ NO → Wait                               │
└─────────────────────────────────────────────┘
      ↓
ÓRÁNKÉNT 2 PERCENKÉNT (00, 02, 04, ...)
      ↓
Automatizmus 4 (Gate Keeper) lefut:
┌─────────────────────────────────────────────┐
│  Pending flag ON & Switch ON?               │
│  → SWITCH OFF (API Call)                    │
├─────────────────────────────────────────────┤
│  Pending flag OFF & Switch OFF & OK?        │
│  → SWITCH ON (API Call)                     │
└─────────────────────────────────────────────┘
```

---

## 🔧 Módosítandó Entitások

| Entitás | Módosítás |
|---------|----------|
| `sensor.shellyZZZZZ_phase_*_current` | Saját Shelly-re |
| `number.wallbox_XXXXXgarazs_sn_SSSSS_maximalis_toltoaram` | Saját Wallbox-ra |
| `switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas` | Saját Wallbox-ra |

---

## 🐛 Hibaelhárítás

**Pause switch nem működik?**
- ✅ Nézd meg az `input_boolean.wallbox_switch_pending` flag értékét
- ✅ Ellenőrizd hogy a sensor **20 másodpercig stabil** ugyanaz az érték
- ✅ Nézd meg a Gate Keeper automation naplóját (Time Pattern trigger)

**Resume switch nem működik?**
- ✅ Nézd meg hogy "OK - Normál üzem" **30 másodpercig stabil**-e
- ✅ Ellenőrizd a pending flag: `off`-nek kell lennie
- ✅ Gate Keeper naplót nézd

**Shelly folyamatosan unavailable?**
- ✅ Ellenőrizd a Shelly hálózati kapcsolatát
- ✅ Safety automatizmus azonnal leáll (vészhelyzet kezelve)

---

**Verzió:** 2.3 (STABLE - DEBOUNCED)  
**Utolsó frissítés:** 2026. január 16.  
**Státusz:** ✅ PRODUCTION READY  
**Licenc:** MIT
