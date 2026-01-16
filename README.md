# 🔌 Wallbox Intelligens Töltésszabályozás v2.3

> **Automatikus, terhelésfüggő Home Assistant töltésszabályozás**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2024.1+-blue)](https://www.home-assistant.io/)
[![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen)](#)
[![Version](https://img.shields.io/badge/Version-2.3%20STABLE-blue)](#)

---

## 🎯 Mit csinál?

Automatikusan szabályozza a Wallbox töltő áramot az aktuális háztartási fogyasztás alapján:

- ⚡ **Valós idejű fázis-monitoring** - Követi az összes 3 fázis áramát
- 🎯 **Terhelésfüggő töltés** - Optimális áram beállítás 2 percenként
- 🛡️ **Biztosíték védelem** - Soha nem lecsap a 16A/fázis limit
- 🚨 **Intelligens szüneteltetés** - Pause/resume az elérható kapacitás alapján
- 🔒 **Rate limiting** - Max 1 API hívás / 120 másodperc (Wallbox API limit-en belül)

---

## 🚀 Telepítés 

### 1️⃣ Helperek Létrehozása

Settings → Devices & Services → Helpers → Create Helper:

```yaml
input_number.wallbox_last_set_current
input_datetime.wallbox_last_switch_time
input_boolean.wallbox_switch_pending
```

### 2️⃣ Template Szenzor

Másold a `wallbox-v2.3-stable.md` dokumentumban található template szenzort a `template.yaml` fájlba.

### 3️⃣ 5 Automatizmus

Importáld az alábbi automatizmusokat (copy-paste az UI-ba vagy YAML-ből):

1. **Wallbox terhelésfüggő töltés** - 2 percenként frissül
2. **Wallbox pause - Request** - Pause flag bejelentkeztetés
3. **Wallbox resume - Request** - Resume flag bejelentkeztetés
4. **Wallbox switch - Gate Keeper** - Központi rate limiter (KRITIKUS!)
5. **Wallbox safety - Shelly unavailable** - Vészhelyzet kezelés

### 4️⃣ Entitások Módosítása

Saját eszközödre módosítsd az alábbi entitásokat:

- `sensor.shellySSSSS_phase_*_current` → Saját Shelly entity
- `number.wallbox_XXXXXgarazs_sn_SSSSS_maximalis_toltoaram` → Saját Wallbox entity
- `switch.wallbox_XXXXXgarazs_sn_SSSSS_szunet_folytatas` → Saját Wallbox switch

### 5️⃣ Teszt

Developer Tools → Template → Próbáld a szenzorok értékeit!

---

## 📊 Hogy Működik?

```
SENSOR ÉRTÉK MEGVÁLTOZIK
      ↓
FIGYELMEZTETÉS (20 sec stabil)? → Pause request
      ↓
OK - NORMÁL ÜZEM (30 sec stabil)? → Resume request
      ↓
ÓRÁNKÉNT 2 PERCENKÉNT
      ↓
GATE KEEPER ellenőrzi:
  • Pending flag ON & Switch ON? → SWITCH OFF
  • Pending flag OFF & Switch OFF & OK? → SWITCH ON
```

---

## 📁 Mit Tartalmaz Ez a Repo?

```
.
├── README.md                      ← Ez a fájl
├── wallbox-v2.3-stable.md        ← Teljes dokumentáció
└── wallbox-v2.3-stable.pdf       ← PDF verzió
```

---

## ❓ Gyakori Kérdések

**Q: Miért nem működik a pause azonnal?**  
A: Szándékosan 20 másodpercet vár a stabilizáció miatt. Ez megakadályozza az ingadozásokat.

**Q: Miért 2 percenként az API hívás?**  
A: Wallbox API limit: 1 req/60 sec. 2 perc = max. 30 call/óra (limit: 60).

**Q: Mi az a "pending flag"?**  
A: Nyomon követi a függőben lévő pause/resume parancsokat az API rate limiting miatt.

---

## 🐛 Hibaelhárítás

**Pause switch nem működik?**
- Nézd meg az `input_boolean.wallbox_switch_pending` flag értékét
- Ellenőrizd hogy a szenzor 20 másodpercig stabil-e
- Nézd meg a Gate Keeper automation naplóját

**Resume switch nem működik?**
- Ellenőrizd hogy "OK - Normál üzem" 30 másodpercig stabil-e
- Nézd meg a pending flag: `off`-nek kell lennie

**Shelly folyamatosan unavailable?**
- Ellenőrizd a Shelly hálózati kapcsolatát
- Safety automatizmus aktiválódik = töltés biztonsági leáll

---

## 📖 Dokumentáció

Teljes dokumentáció a `wallbox-v2.3-stable.md` és `.pdf` fájlban:

- 🔧 Template szenzor részletes magyarázata
- 🤖 5 automatizmus konfigurációja
- 📊 Logika áttekintés
- ⚙️ Konfigurációs lehetőségek
- 🚨 Vészhelyzet kezelés

---

## ⚠️ Disclaimer

- Ezzel a projekttel **saját kockázatra** dolgozol
- **Tesztelve** egy konkrét konfiguráción (Wallbox Pulsar Plus + Shelly 3EM)
- **Saját környezetben** mindig alaposan tesztelj
- A szerzők **nem vesznek felelősséget** az elektromos biztonságért

---

## 📞 Support

Problémák? Nézd meg a teljes dokumentációt vagy nyiss egy Issue-t.

---

```
██╗    ██╗ █████╗ ██╗     ██╗     ██████╗  ██████╗ ██╗  ██╗
██║    ██║██╔══██╗██║     ██║     ██╔══██╗██╔═══██╗╚██╗██╔╝
██║ █╗ ██║███████║██║     ██║     ██████╔╝██║   ██║ ╚███╔╝ 
██║███╗██║██╔══██║██║     ██║     ██╔══██╗██║   ██║ ██╔██╗ 
╚███╔███╔╝██║  ██║███████╗███████╗██████╔╝╚██████╔╝██╔╝ ██╗
 ╚═╝╚═╝╚═╝╚═╝  ╚═╝╚══════╝╚══════╝╚═════╝  ╚═════╝ ╚═╝  ╚═╝
```

**v2.3 STABLE** | Production Ready | MIT License

