# pradcast-homeassistant

Integracja [Home Assistant](https://www.home-assistant.io/) dla **[pradcast.pl](https://pradcast.pl)** - godzinowe ceny prądu w Polsce (RDN/TGE) z prognozą na **D+1..D+3** i gotowymi automatyzacjami do planowania obciążeń: ładowania auta, banku energii, grzania CWU oraz uruchamiania pralki i zmywarki w najtańszych godzinach.

> Faza A: pakiet YAML + blueprinty (instalacja ręczna). Natywna integracja HACS planowana w kolejnym kroku.

---

## Co dostajesz

**Czujniki:**
- `sensor.pradcast_cena_teraz` - cena w bieżącej godzinie (PLN/kWh)
- `binary_sensor.pradcast_tanio_teraz` - czy bieżąca godzina jest tania
- `sensor.pradcast_nastepna_tania_godzina` - najbliższa tania godzina dziś
- `sensor.pradcast_srednia_dzis` / `sensor.pradcast_srednia_jutro` - średnie dobowe
- `sensor.pradcast_raw_dzis` / `sensor.pradcast_raw_jutro` - pełne krzywe 24h (atrybut `prices`)

**Blueprinty (automatyzacje):**
| Plik | Zastosowanie | Tryb |
|---|---|---|
| `boiler_cwu.yaml` | Grzanie CWU/boiler do zadanej godziny | spójne okno N godzin |
| `appliance_run.yaml` | Powiadomienie o najtańszym starcie pralki/zmywarki (+ opcjonalny auto-start) | spójne okno N godzin |
| `ev_charge_deadline.yaml` | Ładowanie auta do godziny wyjazdu | najtańsze rozproszone godziny |
| `battery_arbitrage.yaml` | Ładowanie banku energii w najtańszych godzinach | najtańsze rozproszone godziny |

---

## Wymagania

1. Klucz API z [pradcast.pl/dokumentacja-api](https://pradcast.pl/dokumentacja-api) (darmowy, anonimowy).
2. Home Assistant z dostępem do edycji `configuration.yaml` (Studio Code Server / File editor).
3. Dla wykresu: karta [apexcharts-card](https://github.com/RomRider/apexcharts-card) (przez HACS).

---

## Instalacja

### 1. Pakiet z czujnikami

W `configuration.yaml` włącz katalog pakietów (jeśli jeszcze nie masz):

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Skopiuj [`packages/pradcast.yaml`](packages/pradcast.yaml) do `<config>/packages/pradcast.yaml`.

W `secrets.yaml` dodaj klucz API:

```yaml
pradcast_api_key: pcast_xxxxxxxx
```

Sprawdź konfigurację (**Narzędzia deweloperskie -> Sprawdź konfigurację**) i zrestartuj HA.

### 2. Blueprinty

Kliknij przycisk, by zaimportować blueprint do swojego Home Assistant (wymaga skonfigurowanego [My Home Assistant](https://my.home-assistant.io/)):

| Blueprint | Import |
|---|---|
| Grzanie CWU / boiler | [![Importuj blueprint do Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fpradcast.pl%2Fblueprints%2Fautomation%2Fpradcast%2Fboiler_cwu.yaml) |
| Pralka / zmywarka (powiadomienie) | [![Importuj blueprint do Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fpradcast.pl%2Fblueprints%2Fautomation%2Fpradcast%2Fappliance_run.yaml) |
| Ładowanie auta do wyjazdu | [![Importuj blueprint do Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fpradcast.pl%2Fblueprints%2Fautomation%2Fpradcast%2Fev_charge_deadline.yaml) |
| Ładowanie banku energii | [![Importuj blueprint do Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fpradcast.pl%2Fblueprints%2Fautomation%2Fpradcast%2Fbattery_arbitrage.yaml) |

> Przyciski importują blueprint przez `https://pradcast.pl/blueprints/...` (re-serwowane z tego repo) - dzięki temu w Home Assistant blueprint ląduje w czytelnym folderze `pradcast.pl/`.

Alternatywnie ręcznie: **Ustawienia -> Automatyzacje i sceny -> Blueprinty -> Importuj blueprint**, wklej adres np. `https://pradcast.pl/blueprints/automation/pradcast/boiler_cwu.yaml`.

Po imporcie: **Utwórz automatyzację z blueprinta** i wypełnij pola (encje, godziny, czas pracy).

### 3. Dashboard (opcjonalnie)

Zobacz [`lovelace/example-dashboard.yaml`](lovelace/example-dashboard.yaml).

---

## Jak to działa

Czujniki odpytują publiczne API co 30 min (z dużym zapasem pod limity klucza anonimowego). Bieżąca cena i "tanio teraz" liczone są lokalnie z pobranej krzywej - bez dodatkowych zapytań.

Blueprinty korzystają z endpointu `GET /ha/best-window`, który zwraca najtańsze okno uruchomienia obciążenia przed zadanym terminem:
- **spójne okno** (`interruptible=false`) - N godzin pod rząd, dla cykli AGD i grzania,
- **rozproszone godziny** (`interruptible=true`) - N najtańszych godzin, dla obciążeń przerywalnych (EV, bank).

Okno liczone jest z prognozy **D+1..D+3** sklejonej z aktualnymi cenami RDN, z poziomem pewności (`high`/`medium`/`low`).

## Ograniczenia (Faza A)

- Blueprinty `boiler_cwu` i `appliance_run` czekają (delay) do startu okna - **restart HA w trakcie oczekiwania anuluje** zaplanowane uruchomienie. Blueprinty EV/bank re-planują co godzinę i są na to odporne.
- `ev_charge_deadline` / `battery_arbitrage` zakładają, że ładowarka/falownik zatrzymają ładowanie po osiągnięciu docelowego SoC - "godziny ładowania" to budżet czasu, nie licznik energii.
- Ceny to **RDN (hurt)**, `price_kwh = cena/1000`. Kalkulator ceny detalicznej (taryfy G11/G12, dystrybucja, akcyza, VAT, marża) planowany w kolejnej wersji.

---

## Licencja

[MIT](LICENSE). Dane cenowe pochodzą z publicznego API pradcast.pl (źródło: TGE/ENTSO-E).
