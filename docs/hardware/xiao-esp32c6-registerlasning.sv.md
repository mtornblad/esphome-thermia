# Utökad registerläsning på Thermia Duo / XIAO ESP32-C6

`example-xiao-esp32c6.yaml` inkluderar nu
`thermia-registers-readonly.yaml`. Thread- och PhotoMOS-exemplen ärver detta.
Alla 100 dokumenterade EXT-adresser i ThermIQ-kartan exponeras som läsbara,
diagnostiska sensorer med namn `Register r00` till `Register r74`.
Det odokumenterade intervallet r21–r31 och adresser efter r74 sonderas inte.

På Mattias enhet blir ID:n normalt `sensor.thermia_duo_register_r00` osv.
De befintliga entiteterna och deras namn behålls.

## Vad som faktiskt kan läsas

Pumpmodeller har olika utrustning och firmware. Den här konfigurationen försöker
läsa alla dokumenterade adresser men garanterar inte svar från samtliga på 901510.
Ett uteblivet svar blir unknown/unavailable, inte noll. Ett svar på noll eller
65535 bevaras som råvärde och bevisar inte att en viss funktion finns.

Varje råsensor behåller hela det osignerade 16-bitarsordet, 0–65535. Exempelvis
kan 65535 tolkas som −1 för ett signerat register, men betydelsen ska verifieras
mot pumpens display innan den används för styrning. Särskilt A1/A2, ute-stopps-
kodning och specialräknare ska inte ges en gissad betydelse.
`docs/register-map.json` anger ThermIQ-namn, enheter och källrevision för senare
analys. Detta är en registerkarta, inte verifiering av varje värde på denna pump.

## Vanliga värden prioriteras

De befintliga mätvärdena och exponerade number/select-inställningarna läses i
förgrundsloopen. Efter åtta avslutade förgrundsförfrågningar tas en
bakgrundsförfrågan. Ett uteblivet svar får högst `request_tries` försök; sedan
återgår förfrågningarna till förgrunden. Standard är fortsatt 50 försök eftersom
vissa Thermia-register kräver längre väntan för att ge ett svar.

Råkopior av redan prioriterade register ger inga dubbla förfrågningar. Varje
register lagras en gång i busslagret och kan ha flera presentationsentiteter.
Registret får förgrundsprioritet om någon vanlig entitet använder det.

En första full genomgång kan ta flera minuter, särskilt om många adresser inte
svarar. Sensorerna publiceras med samma 10-sekunders uppdatering som tidigare,
men det betyder inte att varje bakgrundsregister har fått ett nytt svar var
tionde sekund. Råsensorer blir ogiltiga efter 30 minuter utan ett nytt svar;
befintliga vanliga mätvärden behåller sin 120-sekunders gräns.

## Skrivbara inställningar

Utöver befintligt rumsbörvärde och värmekurva samt driftlägesvalet aktiveras:

- Kurvjustering +5, 0 och −5: −5 till +5 °C.
- Värmestopp: 10–25 °C som vald arbetsgräns i HA, inte pumpmenyns fulla gräns.
- Rumfaktor: 0–4; blir relevant när ROOM2/rumsgivaren anslutits.

Att exponera dessa inställningar ändrar inte pumpen vid uppstart. En skrivning
sker först efter ett uttryckligt användarkommando. De 100 råsensorerna har ingen
skrivfunktion, även för adresser som tillhör exempelvis återställningskommandon.
Ingen fabriks-/drifttidsåterställning eller ändring av skyddsgränser automatiseras.

## Bygga och uppdatera från Windows CMD

Efter att ändringen finns på main:

```bat
cd /d C:\temp\esphome-thermia
git pull --ff-only
.venv\Scripts\activate
set "ESPHOME_ESP_IDF_PREFIX=C:\temp\idf"
set "ESPHOME_BUILD_PATH=C:\temp\b"
esphome -s thermia_component_source components compile example-xiao-esp32c6-thread.yaml
```

Behåll dina egna API- och Thread-hemligheter. Använd samma konfigurationsfil som
tidigare; exemplet ovan avser den installerade Thread-varianten.

För uppladdning genom en redan fungerande SSH-tunnel som lyssnar på lokal port 3232:

```bat
esphome -s thermia_component_source components upload example-xiao-esp32c6-thread.yaml --device 127.0.0.1
```

Vid USB-uppladdning anges XIAO:s faktiska COM-port i stället.

## Loggning till BigQuery

Firmwareuppdateringen gör värdena tillgängliga i HA. För att de också ska följa
med i vår externa logg behöver HA-loggpaketet uppdateras enligt separat
`Varme_AI_registerlogg_v1.zip`.

Det paketet lägger råsnapshotar under `pump_registers` i den befintliga
`raw_json`-kolumnen. Befintliga typade BigQuery-fält, buffert, uppladdningskvittens
och databasobjekt behålls; ingen ny initiering av tabellen behövs.

Observerade värdeförändringar i r32–r67 får egna state_change-poster. De befintliga
mätvärdena och nya skrivbara reglagen fortsätter också att loggas. En minutpost
innehåller alla då tillgängliga registervärden. Ändringar som görs och återställs
mellan två registerläsningar kan inte garanteras upptäckas. Tidpunkten är när
interfacet/HA observerade ändringen, inte nödvändigtvis tiden för knapptrycket.
Loggningen avgör inte säkert om ändringen gjordes på pumpen eller via HA.

## Kontroller före leverans

- Bussimulering: 3201 kontroller godkända, inklusive hela 100-registergenomgången,
  förgrundsprioritet, bakgrundstimeout, sena svar, deduplicering och frånvaro av
  automatiska skrivningar.
- XIAO ESP32-C6 Thread kompilerad med ESPHome 2026.10.2 / ESP-IDF 5.5.6.
- Image: 805568 byte, 43,9 % av app-partitionen. RAM: 26,2 %.
- HA-loggning: 37 tester godkända, inklusive råord, negativa tolkningar,
  upptäckt kontra ändring och oförändrat BigQuery-schema.
- Verifiering av faktisk svarsfrekvens och registerstöd på 901510 görs efter
  uppladdningen. Assistenten har inte flashat eller ändrat pumpens driftläge.
