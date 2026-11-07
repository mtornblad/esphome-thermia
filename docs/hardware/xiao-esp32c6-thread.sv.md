# Testa Thermia Duo med XIAO ESP32-C6 över Thread

Använd [example-xiao-esp32c6-thread.yaml](../../example-xiao-esp32c6-thread.yaml).
Den inkluderar EXT-konfigurationen, tar bort WiFi, reserv-AP, captive portal och
WiFi-sensorn och lägger till OpenThread och IPv6. Behåll därför även
`example-xiao-esp32c6.yaml` i samma mapp. Pumpens anslutningar och GPIO22/23 är
oförändrade; [kopplingsguiden](xiao-esp32c6-iso1540.sv.md) gäller fortfarande.

Detta är ESPHome API över Thread. Home Assistant använder ESPHome-integrationen;
ingen Matter-parningskod behövs. Enheten behöver en fungerande **Thread border
router** mellan Thread-meshen och ditt LAN. IPv6 måste fungera mellan Home
Assistant och border routern. Det kräver inte att internetoperatören levererar IPv6.

Thread använder också 2,4 GHz. Ett mesh med en nätdriven Thread-router nära källaren
kan ge en bättre väg, men Thread garanterar inte bättre räckvidd genom betong eller
pumpens metallkapsling. En Zigbee-enhet förlänger inte Thread-nätet.

## Hämta befintliga nätverkets anslutningsuppgifter

Använd hela **Active dataset TLVs / Active Operational Dataset**, en lång hexsträng.
Den innehåller nätverksnyckel, kanal, PAN-ID och andra parametrar i ett paket.
Nätverksnamnet eller bara en 32-teckens nätverksnyckel räcker inte som `tlv`.
Skapa inget nytt Thread-nät för detta test.

### Via Home Assistant

1. Öppna **Inställningar -> Connectivity/Anslutningar -> Thread**. I äldre
   versioner: **Inställningar -> Enheter och tjänster -> Thread -> Konfigurera**.
2. Välj nätverket som din fungerande border router använder och öppna
   **Nätverksinformation** eller informationssymbolen.
3. Om **Active dataset TLVs** visas: kopiera hela hexsträngen.

Home Assistant kan upptäcka ett Thread-nät utan att ha dess hemliga uppgifter.
För ett Google-/Apple-nät kan uppgifterna först behöva skickas från mobilens
Home Assistant Companion-app: öppna Thread-sidan där och välj **Send credentials
to Home Assistant / Skicka autentiseringsuppgifter till Home Assistant**.
Detta garanterar inte att alla versioner visar själva TLV-strängen i dialogen.
Om fältet saknas behöver exportvägen anpassas till din border router och HA-version.

### Via en egen OpenThread Border Router

På en Linuxmaskin som själv kör OTBR, med dess CLI installerat:

```sh
sudo ot-ctl dataset active -x
```

I en OTBR-container är motsvarande kommando `ot-ctl dataset active -x` inne i
**den containern**. Det finns inte nödvändigtvis i Home Assistants vanliga
Terminal-app och ska inte köras i Windows CMD.

På Espressifs ESP Thread Border Router-firmware kan du i dess seriella CLI köra:

```text
ot dataset active -x
```

I ett rent OpenThread CLI utan `ot`-prefix är kommandot `dataset active -x`.
Kopiera bara hexsträngen, inte prompten eller raden `Done`.

## Spara lokalt och bygg

Lägg till i din befintliga, git-ignorerade `secrets.yaml`:

```yaml
thread_dataset_tlv: "DIN_FULLSTANDIGA_HEXSTRANG"
```

Behåll din befintliga `api_encryption_key`: OTA återanvänder samma nyckel.
`ota_password` får stå kvar men används inte av de aktuella C6-exemplen.
WiFi-fälten får stå kvar i
`secrets.yaml`, men används inte av Thread-bygget. Datasetet är en hemlighet:
publicera det inte i GitHub, skärmbilder eller chatten. Exempelvärdet i
`secrets.example.yaml` måste ersättas; det är avsiktligt ingen fungerande nyckel.

Kör i Windows CMD med din ESPHome-venv aktiverad:

```bat
cd /d C:\temp\esphome-thermia
git pull --ff-only
.venv\Scripts\activate
set "ESPHOME_ESP_IDF_PREFIX=C:\temp\idf"
set "ESPHOME_BUILD_PATH=C:\temp\b"
esphome -s thermia_component_source components config example-xiao-esp32c6-thread.yaml
esphome -s thermia_component_source components run example-xiao-esp32c6-thread.yaml --device COMxx
```

Ersätt `COMxx` med XIAO:s verkliga USB-port. Vid första nätverkstestet, lämna
pumpens EXT frånkopplad och kontrollera först att XIAO kan ansluta till Thread.
USB-loggning är kvar och fungerar även om Thread eller IPv6 inte fungerar.
Behöver du bara bygga: `esphome compile example-xiao-esp32c6-thread.yaml`.

`force_dataset: true` gör att XIAO använder datasetet i YAML vid varje uppstart
även om ett tidigare dataset finns sparat på enheten. Det ändrar inte border
routerns inställningar. `FTD` används eftersom XIAO är kontinuerligt USB-matad;
ingen djupsömn konfigureras.

## Kontrollera anslutningen

```bat
esphome logs example-xiao-esp32c6-thread.yaml --device COMxx
```

Thread-rollen ska bli `child`, `router` eller `leader`, inte förbli `detached`.
Rollen `leader` ensam bevisar inte kontakt med ditt befintliga nät; kontrollera
nätverksnamnet och att Home Assistant når ESPHome API på enhetens IPv6-adress.
Konfigurationen visar roll, IPv6, nätverksnamn, anslutningsförsök och omsändningar.
Parent-RSSI och parent-länkkvalitet är mest användbara när enheten har rollen
`child`; de kan vara otillgängliga när enheten själv blivit router.

Home Assistant kan upptäcka enheten via border routerns tjänsteannonsering.
Om den inte gör det: lägg till via **ESPHome-integrationen** med en routbar
Thread-IPv6-adress (OMR), inte en `fe80::`-adress eller en mesh-local-adress.
Enheten använder samma ESPHome API-nyckel som WiFi-bygget.

När nätanslutningen är bekräftad, anslut pumpens kablage med pumpströmmen
frånkopplad och prova EXT. Komponenten använder ESPHomes generella nätstatus,
så Thread-anslutning öppnar busshanteringen efter fem sekunder, precis som WiFi
gjorde. Avbrott i Thread-anslutningen pausar busshanteringen.

Om anslutningen bara fungerar nära border routern: prova en placering utanför
metallkapslingen eller en mellanliggande, routerkapabel Thread-enhet på samma
nät. Batteridrivna Thread-givare fungerar normalt inte som routrar.

För att återgå till WiFi, flasha `example-xiao-esp32c6.yaml` via USB igen.

## Uppdatera över Thread utan USB

När enheten redan kör Thread-profilen kan firmware skickas över nätet.
OTA använder TCP-port 3232 och samma krypteringsnyckel som ESPHome API.
Våra C6-exempel kräver nu krypterad OTA, ärvt från basfilen:

```yaml
ota:
  - platform: esphome
    encryption:
```

Behåll samma `api_encryption_key` som den installerade enheten använder.
Enheten som redan kör ESPHome 2026.10.2 med vår API-nyckel erbjuder OTA-kryptering
även med den tidigare `password:`-konfigurationen. Därför kan den uppdateras direkt
med det nya exemplet. Efter uppdateringen ska OTA-loggen visa `Encryption: required`.

Kör från repo-mappen med aktiverad venv och dina korta byggsökvägar:

```bat
git pull --ff-only
set "ESPHOME_ESP_IDF_PREFIX=C:\temp\idf"
set "ESPHOME_BUILD_PATH=C:\temp\b"
esphome -s thermia_component_source components run example-xiao-esp32c6-thread.yaml --device thermia-duo-c6.local
```

Det bygger från komponenten du just hämtade med Git, laddar upp och öppnar loggen.
Du behöver ingen COM-port. Thread har lägre kapacitet än WiFi, så överföringen kan
ta flera minuter. Enheten startar om efter uppdateringen och EXT-länken återansluter.

Om namnet inte fungerar, ersätt `thermia-duo-c6.local` med enhetens routbara
Thread-IPv6-adress (OMR), exempelvis den adress Home Assistant ansluter till.
Ange adressen direkt efter `--device`, utan URL eller hakparenteser. En
`fe80::`-adress eller mesh-local/RLOC-adress med exempelvis `:0:ff:fe00:xxxx`
är inte en lämplig adress från LAN. Det publicerade diagnostikvärdet kan vara en
sådan intern adress; välj därför inte automatiskt det första IPv6-värdet.
Datorn måste ha fungerande IPv6 och en väg via border routern till Thread-nätet.
Att Home Assistant når enheten bevisar inte att Windows-datorn har samma väg.
Vid timeout behöver vi kontrollera adress och IPv6-routning; börja inte med USB.

För annan, äldre firmware: om loggen inte visar `Encryption: offered, plaintext accepted`
eller `Encryption: required`, följ ESPHomes migreringsguide innan bytet. Lägg inte
permanent till `allow_plaintext_upload: true`; det behövs inte för den testade C6-versionen.

## Referenser

- [ESPHome: OpenThread](https://esphome.io/components/openthread/)
- [ESPHome: OTA och migrering till kryptering](https://esphome.io/components/ota/esphome/#enabling-encryption-on-an-existing-device)
- [Home Assistant: Thread och autentiseringsuppgifter](https://www.home-assistant.io/integrations/thread/)
- [OpenThread: exportera aktivt dataset](https://openthread.io/reference/cli/concepts/dataset)
- [Espressif: ESP Thread BR, CLI och dataset](https://docs.espressif.com/projects/esp-thread-br/en/latest/dev-guide/build_and_run.html)
