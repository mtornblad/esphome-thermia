# Välj Thermia Duos driftläge från Home Assistant

XIAO-exemplet exponerar `select.thermia_duo_driftlage` med fem svenska alternativ.
Thread- och PhotoMOS-exemplen ärver samma inställning från
`example-xiao-esp32c6.yaml`. Välj samma exempel som du redan använder.

| Register r33 | Alternativ | Betydelse |
|---|---|---|
| 0 | Av | Driftläge OFF. Detta bryter inte pumpens strömmatning. |
| 1 | Auto | Normal automatisk drift med värmepump och tillåten tillsats. |
| 2 | Endast värmepump | Kompressordrift utan eltillsats. |
| 3 | Endast tillsats | Drift med eltillsats utan kompressorn. |
| 4 | Endast varmvatten | Varmvattenproduktion utan husvärme. |

Pumpens egna temperaturgränser och startvillkor gäller fortfarande. Auto är det
vanliga på-läget; det är inget kommando att tvångsstarta kompressorn.
Värden 5–16 exponeras inte eftersom deras betydelse inte är verifierad.

## Uppdatera firmware på befintlig Thread-enhet

I Windows CMD, med befintliga hemligheter kvar:

```bat
cd /d C:\temp\esphome-thermia
git pull --ff-only
.venv\Scripts\activate
set "ESPHOME_ESP_IDF_PREFIX=C:\temp\idf"
set "ESPHOME_BUILD_PATH=C:\temp\b"
esphome -s thermia_component_source components compile example-xiao-esp32c6-thread.yaml
```

Den lokala komponentkällan `components` använder koden du just hämtade, så en
äldre cache av GitHub-komponenten inte hindrar den nya `option_labels`-inställningen.

Ladda upp via den OTA-väg du redan har fått att fungera. Om SSH-tunneln lyssnar
lokalt på port 3232, kör i samma aktiverade venv:

```bat
esphome -s thermia_component_source components upload example-xiao-esp32c6-thread.yaml --device 127.0.0.1
```

Använder du USB anger du dess faktiska COM-port i stället.
Firmwareuppdateringen skriver inget nytt driftläge till pumpen vid uppstart.
Efter att HA återanslutit ska lägesväljaren visa pumpens inlästa läge. Kontrollera
först att det överensstämmer med displayen och prova därefter manuellt.

## Lägg till ett vanligt HA-kort

Lägg följande som ett manuellt kort i kontrollpanelen, eller använd de kompletta
panelerna från den separata panelleveransen:

```yaml
type: entities
title: Pumpens driftläge
show_header_toggle: false
entities:
  - entity: select.thermia_duo_driftlage
    name: Driftläge
```

Det krävs ingen HA-hjälpare eller extra integration för lägesvalet. Om du döpt om
enheten i HA kan det faktiska entitets-ID:t skilja sig; välj då dess Driftlage-entitet.

## Vad ett valt läge bekräftar

Komponenten använder befintlig skrivkö till r33. UI uppdateras först optimistiskt
och korrigeras av nästa inläsning. Det finns ingen separat protokollkvittens för
skrivningen. Kontrollera pumpens display efter första lägesändringen.

AV och VARMVATTEN ger ingen normal uppvärmning av huset. Endast värmepump
begränsar eltillsatsen och kan påverka toppvärmeladdningen av varmvattnet enligt
pumpens modell och inställningar. Lägena är manuella val, inte nya automationer.

## Valfria egna etiketter

`option_labels` ändrar endast visningstexten, inte registervärdena. Exakt fem
unika, icke-tomma etiketter krävs i ordningen Av, Auto, Värmepump, Tillsats,
Varmvatten. Utelämnas inställningen används de tidigare engelska etiketterna,
så övriga befintliga exempel behåller sina alternativ.

Källor: ThermIQs `main_mode`/`mode0`–`mode4` i
[registerdefinitionerna](https://github.com/ThermIQ/thermiq_mqtt-ha/blob/master/custom_components/thermiq_mqtt/heatpump/thermiq_regs.py),
och Thermias [skötselanvisning för Villa, avsnitt 3.2.1](https://jseducation.se/files/pages/thermia-villa-anv.pdf).
Mappningen följer det befintliga interfacet; alla fem lägen är ännu inte
hårdvaruprovade på Mattias styrkort 901510.
