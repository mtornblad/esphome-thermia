# Thermia Duo 901510: XIAO ESP32-C6 och ISO1540

## Börja med EXT

Använd [example-xiao-esp32c6.yaml](../../example-xiao-esp32c6.yaml) för
XIAO ESP32-C6 och din befintliga Adafruit ISO1540. XIAO får ström via USB-C.
Inga PhotoMOS-reläer behövs för att läsa EXT eller ändra dess register.
Rumsemulering, nattsänkning och EVU är separata kretsar, beskrivna längre ned.

Detta är C6-konfigurationen för Mattias Thermia Duo med styrkort 901510.
Programvaran måste fortfarande provas mot den fysiska pumpen. Registervärden
ska jämföras med pumpens display innan de används för automatisk styrning.

## Koppla XIAO till isolatorn

På dina bilder är texten **ISO154x upp och ned**. Med bilderna i den orienteringen:

- Isolatorraden närmast de lösa färgade ledningarna är **sida 2, mot XIAO**.
- Raden närmast pumpkabeln är **sida 1, mot pumpen**.

Om du vänder modulen så ISO154x läses rättvänt blir sida 1 den nedre raden,
vilket är samma orientering som i Adafruits dokumentation. Pumpen är I2C-master;
XIAO emulerar en slav. Följ **märkningen vid varje lödpunkt**, inte ledarfärgen.
Ledningarnas koppling på undersidan av ditt experimentkort kan inte verifieras
från bilderna.

| ISO1540 sida 2, mot XIAO | XIAO-kontakt | GPIO i YAML |
|---|---|---|
| VCC | **3V3** | - |
| GND | **GND** | - |
| SDA | **D4** | **GPIO22** |
| SCL | **D5** | **GPIO23** |

Håll XIAO med **USB-C uppåt och metallskölden mot dig**, som på din första bild:

| Kontakt | Fysisk position i den orienteringen |
|---|---|
| D4 / GPIO22 | Vänster rad, femte stiftet uppifrån |
| D5 / GPIO23 | Vänster rad, sjätte stiftet uppifrån |
| GND | Höger rad, andra stiftet uppifrån |
| 3V3 | Höger rad, tredje stiftet uppifrån |

Kontrollera mot kortets märkning/pinout innan du kopplar. D4 betyder **inte** GPIO4.

## Koppla pumpen till isolatorn

| Pumpens EXT-signal | ISO1540 sida 1, mot pumpen |
|---|---|
| +5 V | VCC |
| GND | GND |
| SDA | SDA |
| SCL | SCL |

**Även pumpsidan av isolatorn måste få +5 V och pump-GND.** En kabel med bara
GND/SDA/SCL räcker inte för att försörja den sidan. På bilderna syns inte alla
ledare och anslutningar tydligt; verifiera samtliga fyra funktioner.

EXT-kontaktens fysiska stiftordning måste kontrolleras på ditt 901510-kort.
Originalprojektets bild gäller ett annat dokumenterat Duo-kort och är inte
bevis för stiftordningen på ditt exemplar.

**Koppla inte ihop jordarna över isolationsgränsen.** Pump-GND går bara till
ISO1540 sida 1; XIAO-GND går till sida 2. Pumpens +5 V ska inte gå till XIAO:s
5V-stift i detta USB-matade utförande.

ISO1540-kortet har redan 10 kΩ pull-up på SDA/SCL på båda sidor. På ditt
experimentkort syns även två extra motstånd. Om de är pull-up på XIAO-sidan
måste de gå till **XIAO-sidans 3,3 V**, inte till 5 V eller över isolationsgränsen.
Lägg inte till fler pull-up innan de befintliga anslutningarna är kontrollerade.

Med pump och USB avstängda: kontrollera med multimeter att ingen egen ledning,
lödning eller kopparbana förbinder de två jordarna. Med endast USB på ska
XIAO-sidans VCC vara cirka 3,3 V. Med pumpmatningen på ska pumpsidans VCC
vara cirka 5 V. Mät varje spänning mot **den sidans egen GND**.

## Bygg och programmera från Windows

Klona forken och öppna en terminal i den:

```bat
git clone https://github.com/mtornblad/esphome-thermia.git
cd esphome-thermia
python -m venv .venv
.venv\Scripts\activate
python -m pip install esphome==2026.10.2
copy secrets.example.yaml secrets.yaml
```

Om du redan har en aktiverad ESPHome-miljö kan du använda den. Fyll i `secrets.yaml`:
WiFi-namn, WiFi-lösenord, API-nyckel, OTA-lösenord och lösenord till reservnätet.
`api_encryption_key` måste vara en giltig Base64-kodad 32-byte-nyckel. Skapa en med:

```bat
python -c "import base64,secrets; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

1. Låt pumpens EXT-kabel vara frånkopplad vid den första programmeringen.
2. Anslut XIAO till datorn med en USB-datakabel.
3. Hitta dess COM-port med kommandot nedan. Välj XIAO:s port, inte de äldre
   Bluetooth- eller nRF-portarna.

```bat
python -m serial.tools.list_ports -v
esphome config example-xiao-esp32c6.yaml
esphome run example-xiao-esp32c6.yaml --device COMxx
```

Ersätt `COMxx` med den verkliga porten. Om uppladdningen inte hittar bootloadern:
håll **BOOT**, tryck och släpp **RESET**, släpp sedan **BOOT** och prova igen.
C6 programmeras via ESPHome/esptool, inte genom kopiering av en nRF-UF2-fil.
Tryck RESET efter första flashningen om programmet inte startar automatiskt.

YAML använder USB_SERIAL_JTAG för USB-loggning och väljer den inbyggda antennen.
Du behöver inte koppla någon extern antenn för första testet. Vid drift i pumpens
metallkapsling kan radiotäckningen behöva förbättras senare.

## Första kommunikationen

När USB-loggen fungerar och kablaget är kontrollerat, bryt pumpens ström,
anslut EXT och starta pumpen igen. Kontrollera medvetet att rätt sida av
isolatorn har rätt matning innan SDA/SCL ansluts till XIAO.

```bat
esphome logs example-xiao-esp32c6.yaml --device COMxx
```

- När WiFi varit anslutet cirka fem sekunder börjar komponenten hantera EXT.
- Leta efter `Network is up - starting to talk to the heat pump`, därefter
  `Heat pump is talking to us` och stigande `pings`, `data` och `reads`.
- Vissa register tar längre tid att få svar på; alla värden behöver inte dyka upp direkt.
- Om `pings=0`: kontrollera båda matningarna, sida 1/2 och SDA/SCL.
- Om felen ökar: kontrollera att SDA/SCL inte är växlade och att XIAO-sidan inte har 5 V pull-up.
- För kortvarig felsökning: sätt `debug_frames: true`, bygg och ladda upp igen.
  `sniff: true` är en annan diagnos som stänger av det normala protokollet;
  låt den vara `false` under vanlig provning. Ändra inte NODE som felsökning
  utan skäl; den är redan inställd för ditt tidigare fungerande interface.

Lägg sedan till enheten med ESPHome-integrationen i Home Assistant och dess
API-nyckel. Konfigurationen exponerar temperaturer, status, larm, drifttider,
rumsvärde utan rumsgivare (16-26 °C) och värmekurva (22-56). De två inställningarna
skriver bara efter ett uttryckligt kommando; det finns inga automatiska ändringar
av pumpens inställningar vid uppstart. Undvik att ändra dem innan avläsningen är bekräftad.

Detta är en **WiFi-konfiguration**. Nuvarande komponent pausar EXT när WiFi
kopplas ned för att låta C6 återansluta. Home Assistant-frånkoppling ensam
stoppar inte EXT. Fristående elprisreglering och en Thread-profil ingår inte i detta första test.

## Valfritt: PhotoMOS, rum och EVU

När EXT fungerar och de separata PhotoMOS-kretsarna är byggda kan du använda
[example-xiao-esp32c6-photomos.yaml](../../example-xiao-esp32c6-photomos.yaml).
Behåll båda YAML-filerna i samma mapp; den senare inkluderar den första.

| Funktion | XIAO | GPIO | Relä |
|---|---|---|---|
| Rumsemulering | D0 | 0 | U1 |
| Nattsänkning | D1 | 1 | U2 |
| EVU-spärr | D3 | 21 | U3 |

Varje ingång kopplas likadant: GPIO -> **220 Ω** -> AQY212EH **pinne 1 (+)**,
AQY212EH **pinne 2 (-)** -> XIAO-GND. Sätt **100 kΩ mellan GPIO och XIAO-GND**.
Pinne 3/4 är den isolerade kontakten; pumpens ledningar får inte förbindas med XIAO-GND.

- **U1:** 303 -> **1,1 kΩ last** -> pinne 3; pinne 4 -> 304. Gör lasten av fyra
  parallella grenar där varje gren har två **2,2 kΩ / 0,25 W i serie**: åtta
  motstånd totalt. Ingen gren får kringgå lasten. Ingen kortslutning av 303/304.
- **U2:** 307 -> **10 kΩ / 0,25 W** -> pinne 3; pinne 4 -> 308.
- **U3:** 307 -> pinne 3; pinne 4 -> 308. U2 och U3 ligger alltså parallellt
  över samma plintpar men bara U2-grenen innehåller 10 kΩ.

303/304 och 307/308 är pumpfamiljens dokumenterade plintnummer och måste
verifieras mot schemat i din pump. Kopplingens motståndslaster dimensioneras
för högst **30 V DC** här. Vid 27 V över rumslasten blir effekten cirka 0,66 W
totalt, eller 0,083 W per motstånd. Behåll den ordinarie utegivaren.

```bat
esphome run example-xiao-esp32c6-photomos.yaml --device COMxx
```

Alla tre funktioner startar avstängda. Nattsänkning och EVU har en mjukvaruförregling
och 100 ms växlingstid. Första rumstestet är manuellt: ange den **verkliga uppmätta
rumstemperaturen** i `Rumstemperatur till rumsemulator`, välj ett rimligt börvärde
och slå sedan på `Rumsemulator sandning`. Kontrollera rumsavläsningen på pumpen.
Upstream lagrar temperaturvärden; denna exempelkonfiguration lägger inte till
någon sensor eller timeout för gamla mätdata. Lägg till en verklig givare och
en policy för bortfall innan automatisk rumspåverkan tas i drift. Avbruten
rumssändning kan ge rumsgivarlarm och återgång till pumpens eget rumsvärde.

## Referenser

- [Seeed: XIAO ESP32-C6 pinout och antennval](https://wiki.seeedstudio.com/xiao_esp32c6_getting_started/)
- [Adafruit: ISO1540 pinout, sida 1/2 och pull-up](https://learn.adafruit.com/adafruit-iso1540-bidirectional-i2c-isolator/pinouts)
- [Panasonic: AQY212EH datablad](https://mediap.industry.panasonic.eu/assets/download-files/import/semi_eng_ge1a_aqy21_e_dip4.pdf)
- [ThermIQ: installation och pumpanslutningar](https://thermiq.net/ThermIQ_MQTT_Installation.pdf)
- [Din fork av rumsemulatorn](https://github.com/mtornblad/esphome-thermia-room)
