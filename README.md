# Sonoff TRVZB External Temperature Sensor Calibration (interval) / Kalibracja zewnętrznego czujnika temperatury Sonoff TRVZB (interwał)

Home Assistant blueprint that periodically pushes the value of an external
temperature sensor to one or more Sonoff TRVZB thermostatic radiator valves,
so the TRV regulates against an accurate room temperature instead of its
internal sensor.

Blueprint do Home Assistant, który cyklicznie wysyła wartość z zewnętrznego
czujnika temperatury do jednej lub wielu głowic Sonoff TRVZB, dzięki czemu
głowica reguluje temperaturę na podstawie dokładnego pomiaru z pokoju
zamiast wewnętrznego czujnika.

---

## 🇬🇧 English

### One-click install

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fshaggee%2FSonoff-TRVZB-external-temp-report-and-update-interval%2Fblob%2Fmain%2Fsonoff-trvzb-blueprint.yaml)

Or manually: copy the URL of [`sonoff-trvzb-blueprint.yaml`](./sonoff-trvzb-blueprint.yaml),
then in Home Assistant go to **Settings → Automations & Scenes → Blueprints →
Import Blueprint** and paste it.

### What it does

The Sonoff TRVZB has a built-in option to use an external temperature sensor
in place of its own. To make this work, the TRV must receive the external
temperature reading via Zigbee on a regular basis (otherwise it falls back to
the internal sensor — see *Fail-safe mode* below).

This blueprint:

- Reads a temperature sensor entity in Home Assistant (any `sensor` with
  `device_class: temperature` — Zigbee, Wi-Fi, Bluetooth, calculated, doesn't
  matter).
- Writes that value into the TRV's external-temperature input entity at a
  configurable interval (default: every 5 minutes).
- Supports multiple TRVs in a single instance, with a configurable per-TRV
  delay so the writes don't burst the Zigbee mesh.
- Supports multiple blueprint instances running in parallel, with a
  deterministic per-instance start-second offset so they never collide.

### Features

- ⏱ **Configurable update interval** — every 1 / 2 / 5 / 10 / 15 / 20 / 30
  minutes or every hour.
- 🎯 **Deterministic multi-instance scheduling** — set a different
  `start_offset_seconds` per instance (0–59) and they will fire at different
  seconds of the same minute. No randomness, no collisions.
- 🐌 **Per-TRV delay** — short pause between consecutive TRV writes within a
  single instance.
- 🔄 **Auto-ensure external sensor (v1.6+)** — before each cycle, the
  blueprint checks the TRV's `temperature_sensor_select` entity and switches
  it to `external` if it has drifted (e.g. after Z2M restart or fail-safe).
  Configurable post-switch delay (default 3 s).
- 🌡 **Sensor-agnostic** — Zigbee, Wi-Fi, Bluetooth, template — anything that
  exposes `device_class: temperature`.
- 🛡 **Fail-safe friendly** — interval defaults stay well below the TRV's
  2-hour fail-safe window.
- 🌍 **Bilingual UI** — English and Polish labels, descriptions and option
  names side by side.

### Requirements

- **Sonoff TRVZB firmware**: 1.2.1 or newer.
- **Zigbee integration that exposes the TRV external-temperature input
  entity**, e.g. Zigbee2MQTT v2.1.2+.
- The TRV must be **explicitly configured to use an external temperature
  sensor** (option in the Z2M device page or equivalent).
- Home Assistant 2024.6+ (uses `time_pattern` trigger and `repeat.for_each`).

### Setup walkthrough

1. **Configure the TRV.** In Zigbee2MQTT, open the TRV device and set
   *External temperature sensor* to enabled.

2. **(Recommended) Force-update the external sensor.** By default Z2M does not
   re-publish a sensor reading to HA if the value hasn't changed. The TRV
   needs regular updates even during constant temperature. Edit
   `devices.yaml` in Z2M:

   ```yaml
   '0xd44867fffe111ce1':
     friendly_name: Kitchen Temperature Sensor
     homeassistant:
       temperature:
         force_update: true
   ```

3. **Import the blueprint** using the badge above.

4. **Create an automation** from the blueprint per room (or per group of TRVs
   sharing one sensor):
   - Pick the **External Temperature Sensor**.
   - Pick one or more **Sonoff TRVZB(s)**.
   - Choose **Update Frequency** (default: every 5 minutes).
   - If you have multiple instances of this blueprint running, set
     **Start second offset** to a different value per instance — see below.
   - Optionally adjust **Delay between TRVs** (default: 2 s).
   - Leave **Ensure TRV uses external sensor** on (default) — recommended
     for default Z2M naming. Turn off if you have renamed entities so that
     the auto-derived `select.<id>_temperature_sensor_select` no longer
     matches your installation.
   - Optionally adjust **Delay after switching to external** (default: 3 s).

### Multi-instance scheduling — avoiding Zigbee congestion

Because `time_pattern` fires on the wall clock, by default every instance
fires at the exact same second (e.g. `12:05:00`). With several instances and
multiple TRVs each, you can saturate the Zigbee coordinator for a few
seconds.

The blueprint solves this with a **per-instance start-second offset** —
each instance fires at a different second within the minute. Recommended
offsets:

| Instances | Offsets |
|-----------|------------------|
| 2 | 0, 30 |
| 3 | 0, 20, 40 |
| 4 | 0, 15, 30, 45 |
| 5 | 0, 12, 24, 36, 48 |
| 6 | 0, 10, 20, 30, 40, 50 |

This is deterministic — no randomness, no collisions. The offset works
together with **Delay between TRVs** so a single instance with several
heads also spreads its writes.

### Fail-safe mode

The TRVZB falls back to its internal sensor if it doesn't receive an
external temperature for **2 hours**. This protects against frozen pipes
and overheating when the external sensor goes offline. Once the next
update arrives, it switches back to the external sensor automatically.

Choose an update interval well below 2 hours. Recommended: **5–15 minutes**.

### Reporting tuning (Zigbee2MQTT)

Two settings on the *external sensor* are worth checking:

- **Min Rep Change** — minimum value change to broadcast. `100` typically
  means 1 °C; lowering to `50` (0.5 °C) gives more accurate tracking.
- **Max Rep Interval** — maximum time between broadcasts even with no value
  change. Keep below `7200` (2 h) so the device keeps reporting during long
  constant-temperature periods.

### Auto-ensure external sensor — how it works

The blueprint reads each TRV's `<id>_temperature_sensor_select` entity at
the start of every cycle. If its state isn't `external`, the blueprint
flips it via `select.select_option`, waits the configured delay (3 s by
default), and only then pushes the temperature value. This protects
against silent reverts to the internal sensor after Z2M restart, fail-safe
trigger, or accidental manual change.

The select entity_id is auto-derived from the TRV number entity_id:

```
number.<id>_external_temperature_input  →  select.<id>_temperature_sensor_select
```

E.g. `number.0x44e2f8fffe1a11f6_external_temperature_input` →
`select.0x44e2f8fffe1a11f6_temperature_sensor_select`. This works for both
IEEE-address and friendly-name styles.

If you have renamed the select entity manually, turn the toggle off — the
blueprint will then fall back to "push temperature only".

### Troubleshooting

- **All instances fire in the same second.** You haven't set
  `start_offset_seconds` per instance — see the table above.
- **TRV reverts to internal sensor.** External temperature hasn't been
  written for 2 h. Check the automation is enabled, the sensor is online,
  and `force_update: true` is set in `devices.yaml` if using Z2M. With
  v1.6+, also confirm **Ensure TRV uses external sensor** is on.
- **Auto-ensure does nothing.** The auto-derived select entity_id may not
  match your installation. In Developer Tools → States, look up your TRV's
  `temperature_sensor_select` entity and check whether the id matches the
  pattern `select.<id>_temperature_sensor_select` where `<id>` is identical
  to the one in your `number.<id>_external_temperature_input`. If not,
  rename the entity to match Z2M defaults or turn the toggle off.
- **Zigbee network stuttering.** Increase **Delay between TRVs** to 3–5 s,
  or split TRVs into more instances with different start offsets.

---

## 🇵🇱 Polski

### Instalacja jednym kliknięciem

[![Otwórz swoją instancję Home Assistant i pokaż dialog importu blueprintu z wypełnionym URL-em.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fshaggee%2FSonoff-TRVZB-external-temp-report-and-update-interval%2Fblob%2Fmain%2Fsonoff-trvzb-blueprint.yaml)

Albo ręcznie: skopiuj URL pliku [`sonoff-trvzb-blueprint.yaml`](./sonoff-trvzb-blueprint.yaml),
a następnie w Home Assistant przejdź do **Ustawienia → Automatyzacje
i sceny → Blueprinty → Importuj blueprint** i wklej go.

### Co to robi

Sonoff TRVZB ma wbudowaną opcję używania zewnętrznego czujnika temperatury
zamiast własnego. Aby to działało, głowica musi regularnie otrzymywać
odczyt z zewnętrznego czujnika przez Zigbee (w przeciwnym razie wraca do
wewnętrznego czujnika — patrz *Tryb fail-safe* poniżej).

Ten blueprint:

- Czyta encję czujnika temperatury w Home Assistant (dowolny `sensor` z
  `device_class: temperature` — Zigbee, Wi-Fi, Bluetooth, template, bez
  znaczenia).
- Zapisuje tę wartość do encji wejścia temperatury zewnętrznej w głowicy
  z konfigurowalnym interwałem (domyślnie co 5 minut).
- Obsługuje wiele głowic w jednej instancji, z konfigurowalnym opóźnieniem
  pomiędzy zapisami, żeby nie zatkać sieci Zigbee.
- Obsługuje wiele instancji blueprintu jednocześnie, z deterministycznym
  offsetem sekundy startu per instancja, dzięki czemu nigdy nie kolidują.

### Funkcje

- ⏱ **Konfigurowalny interwał aktualizacji** — co 1 / 2 / 5 / 10 / 15 / 20 / 30
  minut albo co godzinę.
- 🎯 **Deterministyczne harmonogramowanie wielu instancji** — ustaw inny
  `start_offset_seconds` w każdej instancji (0–59), a wyzwolą się w różnych
  sekundach tej samej minuty. Bez losowości, bez kolizji.
- 🐌 **Opóźnienie między TRV** — krótka pauza między kolejnymi zapisami
  w jednej instancji.
- 🔄 **Auto-pilnowanie czujnika zewnętrznego (v1.6+)** — przed każdym cyklem
  blueprint sprawdza encję `temperature_sensor_select` głowicy i przełącza
  ją na `external`, jeśli się przestawiła (np. po restarcie Z2M lub
  fail-safe). Konfigurowalne opóźnienie po przełączeniu (domyślnie 3 s).
- 🌡 **Niezależny od typu czujnika** — Zigbee, Wi-Fi, Bluetooth, template
  — cokolwiek z `device_class: temperature`.
- 🛡 **Zgodny z fail-safe** — domyślne interwały są wyraźnie poniżej
  2-godzinnego okna fail-safe TRV.
- 🌍 **Dwujęzyczne UI** — etykiety, opisy i opcje po angielsku i polsku
  obok siebie.

### Wymagania

- **Firmware Sonoff TRVZB**: 1.2.1 lub nowszy.
- **Integracja Zigbee udostępniająca encję wejścia temperatury zewnętrznej
  TRV**, np. Zigbee2MQTT v2.1.2+.
- Głowica musi być **jawnie skonfigurowana do używania zewnętrznego czujnika
  temperatury** (opcja w karcie urządzenia w Z2M lub ekwiwalent).
- Home Assistant 2024.6+ (używa triggera `time_pattern` i `repeat.for_each`).

### Krok po kroku

1. **Skonfiguruj głowicę.** W Zigbee2MQTT otwórz urządzenie TRV i włącz
   *External temperature sensor*.

2. **(Zalecane) Wymuś aktualizacje czujnika zewnętrznego.** Domyślnie Z2M
   nie publikuje ponownie odczytu do HA, jeśli wartość się nie zmieniła.
   Głowica potrzebuje regularnych aktualizacji nawet podczas stałej
   temperatury. Edytuj `devices.yaml` w Z2M:

   ```yaml
   '0xd44867fffe111ce1':
     friendly_name: Kitchen Temperature Sensor
     homeassistant:
       temperature:
         force_update: true
   ```

3. **Zaimportuj blueprint** używając przycisku powyżej.

4. **Stwórz automatyzację** z blueprintu per pokój (lub per grupa głowic
   współdzielących jeden czujnik):
   - Wybierz **Zewnętrzny czujnik temperatury**.
   - Wybierz jedną lub więcej **Głowic Sonoff TRVZB**.
   - Wybierz **Częstotliwość aktualizacji** (domyślnie: co 5 minut).
   - Jeśli masz wiele instancji tego blueprintu, ustaw **Offset sekundy
     startu** na inną wartość w każdej instancji — patrz niżej.
   - Opcjonalnie dostosuj **Opóźnienie między TRV** (domyślnie: 2 s).
   - Zostaw **Wymuś używanie zewnętrznego czujnika przez TRV** włączone
     (domyślnie) — zalecane przy domyślnym nazewnictwie Z2M. Wyłącz, jeśli
     zmieniłeś nazwy encji i wyliczona automatycznie nazwa
     `select.<id>_temperature_sensor_select` nie odpowiada Twojej instalacji.
   - Opcjonalnie dostosuj **Opóźnienie po przełączeniu na external**
     (domyślnie: 3 s).

### Harmonogramowanie wielu instancji — unikanie zatkania Zigbee

Ponieważ `time_pattern` wyzwala się według zegara, domyślnie wszystkie
instancje odpalają się w tej samej sekundzie (np. `12:05:00`). Przy
kilku instancjach i kilku głowicach każda możesz wysycić koordynator
Zigbee na kilka sekund.

Blueprint rozwiązuje to przez **offset sekundy startu per instancja** —
każda instancja odpala się w innej sekundzie minuty. Zalecane offsety:

| Liczba instancji | Offsety |
|-----------|------------------|
| 2 | 0, 30 |
| 3 | 0, 20, 40 |
| 4 | 0, 15, 30, 45 |
| 5 | 0, 12, 24, 36, 48 |
| 6 | 0, 10, 20, 30, 40, 50 |

To jest deterministyczne — bez losowania, bez kolizji. Offset współpracuje
z **Opóźnieniem między TRV**, więc pojedyncza instancja z kilkoma
głowicami też rozkłada swoje zapisy.

### Tryb fail-safe

TRVZB wraca do wewnętrznego czujnika, jeśli nie otrzyma temperatury
zewnętrznej przez **2 godziny**. To zabezpiecza przed zamarzaniem rur
i przegrzaniem, gdy zewnętrzny czujnik zniknie. Po otrzymaniu kolejnej
aktualizacji automatycznie wraca do czujnika zewnętrznego.

Wybierz interwał wyraźnie poniżej 2 godzin. Zalecane: **5–15 minut**.

### Strojenie raportowania (Zigbee2MQTT)

Warto sprawdzić dwa ustawienia *czujnika zewnętrznego*:

- **Min Rep Change** — minimalna zmiana wartości potrzebna do rozesłania
  raportu. `100` zwykle oznacza 1 °C; zmniejszenie do `50` (0,5 °C) daje
  dokładniejsze podążanie za temperaturą.
- **Max Rep Interval** — maksymalny czas między raportami nawet bez zmiany
  wartości. Trzymaj poniżej `7200` (2 h), aby urządzenie raportowało
  podczas długich okresów stałej temperatury.

### Auto-pilnowanie czujnika zewnętrznego — jak to działa

Blueprint odczytuje encję `<id>_temperature_sensor_select` każdej głowicy
na początku każdego cyklu. Jeśli jej stan to nie `external`, blueprint
przełącza ją przez `select.select_option`, czeka skonfigurowane opóźnienie
(domyślnie 3 s) i dopiero wtedy wysyła wartość temperatury. To zabezpiecza
przed cichym powrotem do wewnętrznego czujnika po restarcie Z2M, aktywacji
fail-safe lub przypadkowej zmianie ręcznej.

Encja select jest wyliczana automatycznie z entity_id wejścia temperatury
TRV:

```
number.<id>_external_temperature_input  →  select.<id>_temperature_sensor_select
```

Np. `number.0x44e2f8fffe1a11f6_external_temperature_input` →
`select.0x44e2f8fffe1a11f6_temperature_sensor_select`. Działa zarówno dla
adresu IEEE, jak i friendly name.

Jeśli zmieniłeś nazwę encji select ręcznie, wyłącz tę opcję — blueprint
wróci do trybu "tylko wysyłaj temperaturę".

### Rozwiązywanie problemów

- **Wszystkie instancje odpalają się w tej samej sekundzie.** Nie ustawiłeś
  `start_offset_seconds` per instancja — patrz tabela wyżej.
- **TRV wraca do wewnętrznego czujnika.** Temperatura zewnętrzna nie była
  zapisana przez 2 h. Sprawdź, czy automatyzacja jest włączona, czy czujnik
  jest online, i czy `force_update: true` jest ustawione w `devices.yaml`
  jeśli używasz Z2M. W v1.6+ sprawdź też, czy **Wymuś używanie zewnętrznego
  czujnika przez TRV** jest włączone.
- **Auto-pilnowanie nie działa.** Wyliczona automatycznie nazwa encji
  select może nie pasować do Twojej instalacji. W Narzędzia developerskie
  → Stany znajdź encję `temperature_sensor_select` Twojego TRV i sprawdź,
  czy id ma format `select.<id>_temperature_sensor_select`, gdzie `<id>`
  jest takie samo jak w `number.<id>_external_temperature_input`. Jeśli
  nie — zmień nazwę encji do domyślnej Z2M lub wyłącz tę opcję.
- **Sieć Zigbee zacina się.** Zwiększ **Opóźnienie między TRV** do 3–5 s,
  albo rozdziel głowice na więcej instancji z różnymi offsetami startu.

---

## License / Licencja

MIT — see [`LICENSE`](./LICENSE).

## Credits / Podziękowania

Originally based on the v1.0.1 blueprint by the Home Assistant community.
This fork adds periodic interval triggering, deterministic multi-instance
scheduling, per-TRV delay, auto-ensure external sensor (v1.6+), and
bilingual EN/PL documentation.

Pierwotnie oparte na blueprintcie v1.0.1 ze społeczności Home Assistant.
Ten fork dodaje cykliczne wyzwalanie, deterministyczne harmonogramowanie
wielu instancji, opóźnienia między TRV, auto-pilnowanie zewnętrznego
czujnika (v1.6+) oraz dwujęzyczną dokumentację EN/PL.
