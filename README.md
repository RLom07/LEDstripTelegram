# Telegram → NodeMCU: een ledstrip besturen met een Telegram-bot

Door Ronald Lommers · HvA CMD · Internet of Things · ToDo 4
Laatst bijgewerkt: 23 september 2026

![Disco-modus op de ledstrip](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54%20%287%29.jpeg)

## Introductie

In deze manual leer je hoe je een NodeMCU (ESP8266) bestuurt via Telegram. Je maakt een eigen Telegram-bot en stuurt daar berichten naar vanaf je telefoon. De NodeMCU leest die berichten, stuurt een antwoord terug en zet een ledstrip aan, uit of in disco-modus.

Ik volg hiervoor de beschrijving van de docent ([Telegram Adafruit ESP8266](https://icthva.sharepoint.com/:w:/s/FDMCI_ORG__CMD-Amsterdam/Eb7Jd27yWphMuVFbMHV_9WoBEg5_zqAQilsb6Q3gPSKueg?e=f5PM7l)). De focus van deze manual ligt op **wat er misging**. Bij elke fout laat ik zien wat ik zag, wat de oorzaak was, hoe ik daarachter kwam en hoe je het oplost.

### Wat heb je nodig?

1. NodeMCU 1.0 (ESP-12E), in mijn geval een LoLin V3
2. Een NeoPixel-ledstrip (WS2812B) met 16 LEDs, aangesloten met drie draden: data (geel) op **D1**, plus (rood) en GND (zwart)
3. Een micro-USB-kabel
4. [Arduino IDE](https://www.arduino.cc/en/software) (ik gebruik versie 2.3.10) met het ESP8266-board geïnstalleerd
5. Telegram op je telefoon
6. Een **2,4 GHz**-wifinetwerk, bijvoorbeeld de hotspot van je telefoon

### Overzicht van de fouten

| # | Fout | Stap |
|---|------|------|
| 1 | Serial Monitor toont alleen puntjes: wifi verbindt niet | [Stap 4](#stap-4-uploaden-en-wifi-controleren) |
| 2 | Bot stuurt niets terug: tekst op de plek van de parse mode | [Stap 6](#stap-6-een-eigen-antwoord-terugsturen) |
| 3 | Compileerfout `invalid conversion from 'const char*' to 'int'` | [Stap 6](#stap-6-een-eigen-antwoord-terugsturen) |
| 4 | "Unknown command" door een hoofdletter | [Stap 7](#stap-7-de-ingebouwde-led-aansturen-met-commandos) |
| 5 | Het lampje op het board gaat aan, de ledstrip niet | [Stap 8](#stap-8-de-ledstrip-koppelen) |
| 6 | Bot antwoordt, maar de ledstrip doet niets (`digitalWrite` op een NeoPixel) | [Stap 8](#stap-8-de-ledstrip-koppelen) |
| 7 | Disco-modus stopt niet (variabele opnieuw aangemaakt) | [Stap 9](#stap-9-disco-modus) |
| 8 | Disco-animatie beweegt niet (code in de verkeerde functie) | [Stap 9](#stap-9-disco-modus) |

> [!WARNING]
> Je bot-token is een wachtwoord. Iedereen die het token heeft, kan jouw bot besturen. Zet het nooit in een openbare repository en maak het onleesbaar op screenshots. Is dat toch gebeurd? Stuur `/revoke` naar BotFather en gebruik het nieuwe token.

---

## Stap 1: Een bot aanmaken bij BotFather

Installeer Telegram en maak een account aan. Zoek daarna de gebruiker **BotFather** (met het blauwe vinkje) en stuur `/start`. Je krijgt een lijst met alle commando's.

![BotFather start-menu](images/Screenshot%202026-09-23%20190614.png)

Stuur `/newbot`. BotFather vraagt eerst om een naam (ik koos *Callister*) en daarna om een gebruikersnaam die moet eindigen op `bot` (ik koos *Callister_MCU_bot*). Daarna krijg je je **token**. Die heb je zo nodig.

![Een nieuwe bot aanmaken met /newbot](images/Screenshot%202026-09-23%20190019.png)

## Stap 2: Libraries installeren

Open Arduino IDE en ga naar **Tools → Manage Libraries**. Installeer:

1. **UniversalTelegramBot** van Brian Lough. Hiermee praat je NodeMCU met Telegram.

![Library UniversalTelegramBot](images/Screenshot%202026-09-23%20190724.png)

2. **ArduinoJson** van Benoit Blanchon. Neem de laatste versie, niet de beta. Telegram stuurt berichten als JSON en deze library leest die uit.

![Library ArduinoJson](images/Screenshot%202026-09-23%20190820.png)

Later (stap 8) heb je ook **Adafruit NeoPixel** nodig voor de ledstrip. Die kun je nu alvast installeren.

## Stap 3: EchoBot openen en instellen

Ga naar **File → Examples → UniversalTelegramBot → ESP8266 → EchoBot**. Scroll ver naar beneden in de lijst, de voorbeelden van libraries staan onderaan.

Bovenin de code staan placeholders voor je wifi en token:

![Placeholders in EchoBot](images/Screenshot%202026-09-23%20190910.png)

Vul hier je wifinaam, wachtwoord en bot-token in. Let op: de wifinaam moet **exact** kloppen, inclusief hoofdletters en spaties. Hier ging het bij mij mis, zie fout 1.

![Ingevulde gegevens met de fout in de wifinaam](images/Screenshot%202026-09-23%20191552.png)

Kies bovenin het juiste board (**NodeMCU 1.0 (ESP-12E Module)**) en de juiste poort via **Tools → Port**.

![Board kiezen](images/Screenshot%202026-09-23%20191806.png)

![Tools-menu met board en poort](images/Screenshot%202026-09-23%20191938.png)

## Stap 4: Uploaden en wifi controleren

Klik op **Upload** en open daarna **Tools → Serial Monitor**. Zet de baudrate rechtsboven op **115200**, anders zie je onleesbare tekens.

> [!TIP]
> De eerste regels ("Connecting to Wifi SSID…") worden geprint direct na het opstarten, vaak voordat je de Serial Monitor open hebt. Druk op het **RST**-knopje van de NodeMCU om ze alsnog te zien.

### ❌ Fout 1: Alleen puntjes in de Serial Monitor

**Wat zag ik:** een eindeloze rij puntjes, verder niets.

![Serial Monitor met alleen puntjes](images/Screenshot%202026-09-23%20192121.png)

**Oorzaak:** de puntjes komen uit deze loop in `setup()`:

```cpp
while (WiFi.status() != WL_CONNECTED)
{
  Serial.print(".");
  delay(500);
}
```

De NodeMCU blijft proberen verbinding te maken met de wifi en komt nooit verder. Bij mij kwam dat door de wifinaam: ik had `"Iphone (2)"` ingevuld, maar mijn hotspot heet `"iPhone (2)"`. De naam is hoofdlettergevoelig.

**Hoe vind je de oorzaak?** Print de wifistatus in plaats van alleen een punt:

```cpp
while (WiFi.status() != WL_CONNECTED)
{
  Serial.print(" status: ");
  Serial.println(WiFi.status());
  delay(500);
}
```

| Status | Betekenis | Mogelijke oorzaak |
|--------|-----------|-------------------|
| `1` | Netwerk niet gevonden | Naam klopt niet, of het is een 5 GHz-netwerk |
| `4` / `6` | Verbinding mislukt / verkeerd wachtwoord | Wachtwoord klopt niet |
| `7` | Nog aan het verbinden | Blijft dit hangen, dan is het meestal ook een netwerkprobleem |

Je kunt ook het voorbeeld **File → Examples → ESP8266WiFi → WiFiScan** uploaden. Dat laat zien welke netwerken de NodeMCU ziet en hoe ze precies heten.

**Oplossingen:**
- Controleer de exacte naam van je netwerk. Op een iPhone: Instellingen → Algemeen → Info → Naam.
- De ESP8266 werkt alleen op **2,4 GHz**. Op een nieuwere iPhone zet je bij Persoonlijke hotspot **"Maximaliseer compatibiliteit"** aan.
- Houd het hotspot-scherm open op je telefoon terwijl de NodeMCU verbindt, anders is de hotspot soms onzichtbaar.

**✅ Resultaat na het aanpassen van de naam:**

![Wifi verbonden](images/Screenshot%202026-09-23%20193001.png)

## Stap 5: Berichten tonen in de Serial Monitor

Stuur `/start` naar je bot in Telegram. De EchoBot stuurt elk bericht precies zo terug.

![De bot echoot berichten](images/WhatsApp%20Image%202026-09-23%20at%2020.22.53%20%281%29.jpeg)

Nu wil je ook op je computer zien wat er binnenkomt. De originele functie `handleNewMessages` stuurt het bericht alleen terug:

![Originele handleNewMessages](images/Screenshot%202026-09-23%20193449.png)

Zet binnen de for-loop een `Serial.println`:

```cpp
void handleNewMessages(int numNewMessages)
{
  for (int i = 0; i < numNewMessages; i++)
  {
    Serial.println(bot.messages[i].text);   // toon het ontvangen bericht
    bot.sendMessage(bot.messages[i].chat_id, bot.messages[i].text, "");
  }
}
```

![Serial.println toegevoegd](images/Screenshot%202026-09-23%20193508.png)

De for-loop gaat langs alle nieuwe berichten. `bot.messages[i].text` is de tekst die jij in Telegram typte.

> [!TIP]
> Zie je wel `got response` maar geen tekst eronder? Dan draait waarschijnlijk nog de oude code op de NodeMCU. Controleer of het uploaden gelukt is en upload opnieuw.
>
> ![Alleen got response](images/Screenshot%202026-09-23%20193620.png)

**✅ Resultaat:** de tekst verschijnt in de Serial Monitor.

![Berichten in de Serial Monitor](images/Screenshot%202026-09-23%20194738.png)

## Stap 6: Een eigen antwoord terugsturen

Volgens de beschrijving van de docent pas je nu het antwoord van de bot aan. Ik wilde dat de bot "Welcome on board captain, all systems online!" terugstuurt.

### ❌ Fout 2: De bot stuurt helemaal niets terug

**Wat zag ik:** de code compileerde en het bericht kwam aan in de Serial Monitor, maar in Telegram kwam geen antwoord.

![Code met de tekst op de verkeerde plek](images/Screenshot%202026-09-23%20193736.png)

![Berichten komen binnen](images/Screenshot%202026-09-23%20194009.png)

![Geen antwoord in Telegram](images/WhatsApp%20Image%202026-09-23%20at%2020.22.53%20%282%29.jpeg)

**Oorzaak:** het derde argument van `sendMessage` is geen tweede bericht, maar de **parse mode**: hoe Telegram de tekst moet opmaken.

```cpp
bot.sendMessage(chat_id, tekst, parse_mode);
```

| Argument | Betekenis |
|----------|-----------|
| `chat_id` | Naar welk gesprek het bericht gaat |
| `tekst` | Het bericht dat de bot stuurt |
| `parse_mode` | `""` (geen opmaak), `"Markdown"` of `"HTML"` |

Ik gaf mijn welkomsttekst mee als parse mode. Die waarde kent Telegram niet, dus het bericht werd niet verstuurd. Je krijgt hier geen foutmelding van: de code compileert gewoon.

**Hoe vind je dit?** `sendMessage` geeft `true` of `false` terug. Print dat:

```cpp
bool gelukt = bot.sendMessage(chat_id, "Hallo!", "");
Serial.println(gelukt ? "Bericht verstuurd" : "Versturen mislukt");
```

### ❌ Fout 3: Compileerfout `invalid conversion from 'const char*' to 'int'`

Mijn eerste poging om fout 2 op te lossen was een extra `""` achteraan:

```cpp
bot.sendMessage(bot.messages[i].chat_id, bot.messages[i].text, "Welcome on board captian, all systems online!", "");
```

**Wat zag ik:**

![Compileerfout](images/Screenshot%202026-09-23%20194538.png)

**Hoe lees je deze foutmelding?**

- De regel met `error:` zegt **wat** er mis is: je probeert tekst (`const char*`) in een getal (`int`) te stoppen.
- Het `^~`-teken wijst aan **waar**: bij de laatste `""`.
- De regel met `note:` laat zien wat de functie **verwacht**:

```cpp
bool sendMessage(const String& chat_id, const String& text, const String& parse_mode = "", int message_id = 0);
```

Het vierde argument is `message_id`: een getal waarmee de bot op een specifiek bericht kan reageren. Daar hoort geen tekst. Bovendien stond mijn welkomsttekst nog steeds op plek 3, dus het echte probleem (fout 2) was niet opgelost.

**✅ Oplossing voor fout 2 en 3:** drie argumenten, met de tekst op plek 2.

```cpp
bot.sendMessage(bot.messages[i].chat_id, "Welcome on board captain, all systems online!", "");
```

![Werkende sendMessage](images/Screenshot%202026-09-23%20194724.png)

![Welkomstbericht in Telegram](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54.jpeg)

> **Verschil tussen fout 2 en 3:** fout 2 is een fout tijdens het draaien. De code compileert, maar doet niet wat je wilt. Fout 3 is een compileerfout. De code kan niet eens geüpload worden, maar de compiler vertelt je precies wat er mis is.

## Stap 7: De ingebouwde LED aansturen met commando's

Stel de ingebouwde LED in als output. Voeg in `setup()` toe:

```cpp
pinMode(LED_BUILTIN, OUTPUT);  // Initialize the LED_BUILTIN pin as an output
```

![pinMode LED_BUILTIN](images/Screenshot%202026-09-23%20194912.png)

Maak daarna in de for-loop van `handleNewMessages` een `if / else if` die naar de tekst luistert:

```cpp
String text = bot.messages[i].text;
String chat_id = bot.messages[i].chat_id;

Serial.println(text);

if (text == "lights on")
{
  digitalWrite(LED_BUILTIN, LOW);   // LED aan
  bot.sendMessage(chat_id, "Lights on, captain!", "");
}
else if (text == "lights off")
{
  digitalWrite(LED_BUILTIN, HIGH);  // LED uit
  bot.sendMessage(chat_id, "Lights off, captain!", "");
}
else
{
  bot.sendMessage(chat_id, "Unknown command. Try 'lights on' or 'lights off'.", "");
}
```

![if / else if met LED_BUILTIN](images/Screenshot%202026-09-23%20195254.png)

> **Let op:** in de beschrijving staat `LED_BUILTIN = LOW`. Dat is geen echte code, maar een omschrijving. In Arduino gebruik je `digitalWrite(LED_BUILTIN, LOW);`.
>
> **Waarom LOW = aan?** De ingebouwde LED van de NodeMCU is *active low*: hij brandt als de pin LOW is. Dat is omgekeerd van wat je zou verwachten.

De laatste `else` staat niet in de opdracht, maar is handig: zo weet je in Telegram altijd of je bericht is aangekomen.

### ❌ Fout 4: "Unknown command" terwijl ik "lights on" typte

**Wat zag ik:**

![Unknown command door een hoofdletter](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54%20%282%29.jpeg)

**Oorzaak:** mijn telefoon maakte er automatisch "**L**ights on" van. `==` is hoofdlettergevoelig, dus "Lights on" is niet gelijk aan "lights on". Toen ik handmatig een kleine letter typte, werkte het wel.

**✅ Oplossing:** zet de tekst eerst om naar kleine letters en haal spaties aan het begin en eind weg. Autocorrect voegt soms een spatie toe.

```cpp
Serial.println(text);
text.toLowerCase();  // "Lights on" wordt "lights on"
text.trim();         // "lights on " wordt "lights on"
```

![toLowerCase en trim](images/Screenshot%202026-09-23%20201040.png)

> **Nog een klassieke fout:** `if (text = "lights on")` met één `=` is geen vergelijking maar een toewijzing. Die is altijd waar. Gebruik altijd `==`.

## Stap 8: De ledstrip koppelen

Mijn ledstrip zit op pin **D1**.

### ❌ Fout 5: Het lampje op het board gaat aan, de strip niet

**Wat zag ik:** de bot antwoordde "Lights on, captain!", maar alleen het kleine blauwe lampje op de NodeMCU ging aan. De strip bleef uit.

![Blauwe LED op het board brandt, strip is uit](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54%20%281%29.jpeg)

**Oorzaak:** `LED_BUILTIN` is het lampje dat op het board zelf zit (GPIO2, pin D4). De code wist niet dat er iets op D1 zat.

**Oplossing:** definieer je eigen pin en gebruik die overal in plaats van `LED_BUILTIN`:

```cpp
#define LED_PIN D1
```

![LED_PIN gedefinieerd](images/Screenshot%202026-09-23%20195806.png)

![pinMode met LED_PIN](images/Screenshot%202026-09-23%20195837.png)

Een externe, gewone LED is meestal **niet** active low. Draai HIGH en LOW dus om: HIGH = aan, LOW = uit.

### ❌ Fout 6: De bot antwoordt, maar de ledstrip doet helemaal niets

**Wat zag ik:** na de aanpassing van fout 5 werkten de commando's in Telegram, maar de strip ging niet aan of uit.

![Commando's werken in Telegram](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54%20%283%29.jpeg)

![Code met digitalWrite op LED_PIN](images/Screenshot%202026-09-23%20195924.png)

**Hoe vond ik de oorzaak?** Dezelfde strip werkte wel in een eerdere opdracht (een weerstation). Door beide stukken code te vergelijken zag ik het verschil. In het weerstation gebruikte ik:

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel pixels(16, D1, NEO_GRB + NEO_KHZ800);
pixels.fill(pixels.Color(255, 0, 0));
pixels.show();
```

**Oorzaak:** mijn strip is een **NeoPixel** (WS2812B). Elke LED heeft een eigen chipje dat een datasignaal verwacht met de kleur per LED. `digitalWrite` zet de pin alleen aan of uit, en dat begrijpt de strip niet. De bedrading en de pin waren dus goed, alleen de manier van aansturen was fout.

| | Gewone LED | NeoPixel-strip |
|---|---|---|
| Draden | 2 (+ en −) | 3 (5V, GND, data) |
| Aansturen | `digitalWrite(pin, HIGH)` | Library: `pixels.fill()` + `pixels.show()` |
| Kleur | Vast | Per LED instelbaar |

**✅ Oplossing:** gebruik de Adafruit NeoPixel-library.

1. Voeg de library toe:

```cpp
#include <Adafruit_NeoPixel.h>
```

![Include NeoPixel](images/Screenshot%202026-09-23%20200908.png)

2. Stel de strip in. `NUMPIXELS` is het aantal LEDs op je strip:

```cpp
#define LED_PIN    D1
#define NUMPIXELS  16
#define HELDERHEID 80
```

![Defines voor de strip](images/Screenshot%202026-09-23%20200933.png)

3. Maak het strip-object aan, onder `bot_lasttime`:

```cpp
Adafruit_NeoPixel pixels(NUMPIXELS, LED_PIN, NEO_GRB + NEO_KHZ800);
```

![Het pixels-object](images/Screenshot%202026-09-23%20201012.png)

4. Vervang in `setup()` de `pinMode` door:

```cpp
pixels.begin();
pixels.setBrightness(HELDERHEID);
pixels.clear();
pixels.show();   // strip uit bij het opstarten
```

![pixels.begin in setup](images/Screenshot%202026-09-23%20201205.png)

5. Vervang in de commando's `digitalWrite` door:

```cpp
if (text == "lights on")
{
  pixels.fill(pixels.Color(255, 255, 255));  // alle LEDs wit
  pixels.show();
  bot.sendMessage(chat_id, "Lights on, captain!", "");
}
else if (text == "lights off")
{
  pixels.clear();
  pixels.show();
  bot.sendMessage(chat_id, "Lights off, captain!", "");
}
```

![NeoPixel aan en uit](images/Screenshot%202026-09-23%20201135.png)

> [!IMPORTANT]
> `fill()` en `clear()` veranderen alleen wat de NodeMCU in zijn geheugen heeft. Pas bij `pixels.show()` wordt dat naar de strip gestuurd. Vergeet je `show()`, dan gebeurt er niets.

**✅ Resultaat:**

| Lights on | Lights off |
|---|---|
| ![Strip aan](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54%20%284%29.jpeg) | ![Strip uit](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54%20%285%29.jpeg) |

## Stap 9: Disco-modus

Disco-modus moet blijven bewegen, ook als er geen nieuwe berichten binnenkomen. Daarom gebruik je een schakelaar (`discoModus`) en een timer.

1. Maak twee globale variabelen aan, onder het `pixels`-object:

```cpp
bool discoModus = false;
unsigned long laatsteDiscoStap = 0;
```

![Disco-variabelen](images/Screenshot%202026-09-23%20201539.png)

2. Voeg een commando toe en pas het "Unknown command"-bericht aan:

```cpp
else if (text == "disco")
{
  discoModus = true;
  bot.sendMessage(chat_id, "Disco mode activated, captain!", "");
}
else
{
  bot.sendMessage(chat_id, "Unknown command. Try 'lights on', 'lights off' or 'disco'.", "");
}
```

![Unknown command aangepast](images/Screenshot%202026-09-23%20201745.png)

3. Zet onderaan in `loop()` de animatie. Elke 100 ms krijgt elke LED een willekeurige, felle kleur:

```cpp
if (discoModus && millis() - laatsteDiscoStap > 100)
{
  laatsteDiscoStap = millis();
  for (int i = 0; i < NUMPIXELS; i++)
  {
    pixels.setPixelColor(i, pixels.ColorHSV(random(65536)));
  }
  pixels.show();
}
```

`ColorHSV` kiest een willekeurige tint op de kleurencirkel, zodat je altijd felle kleuren krijgt en nooit bijna zwart.

### ❌ Fout 7: Disco-modus stopt niet

**Wat stond er in mijn code:** bij "lights on" en "lights off" had ik `bool discoModus = false;` gezet.

![bool discoModus in de if](images/Screenshot%202026-09-23%20201659.png)

**Oorzaak:** door het woord `bool` ervoor maak je een **nieuwe, lokale** variabele aan met toevallig dezelfde naam. Die bestaat alleen binnen dat `if`-blok. De globale `discoModus` bovenaan blijft `true`, dus de disco blijft draaien en overschrijft meteen je "lights on" of "lights off". De compiler geeft hier geen foutmelding over.

**✅ Oplossing:** haal `bool` weg, zodat je de bestaande variabele aanpast:

```cpp
discoModus = false;
```

### ❌ Fout 8: De disco-animatie beweegt niet

**Wat stond er in mijn code:** het animatieblok stond binnen de for-loop van `handleNewMessages`, in plaats van in `loop()`. Je ziet het aan de inspringing.

![Disco-code in de verkeerde functie](images/Screenshot%202026-09-23%20201832.png)

**Oorzaak:** `handleNewMessages` wordt alleen aangeroepen als er een nieuw bericht binnenkomt. De kleuren veranderen dan maar één keer per bericht, in plaats van continu.

**✅ Oplossing:** knip het blok uit en plak het onderaan in `loop()`, na het blok dat berichten ophaalt.

> **Waarom `millis()` en geen `delay()`?** Met `while (true) { ...; delay(100); }` werkt de disco wel, maar komt je code nooit meer bij `bot.getUpdates()`. De bot reageert dan niet meer op "lights off". `millis()` kijkt alleen hoeveel tijd er verstreken is, zonder de rest van de code te blokkeren.
>
> **Bekende beperking:** de animatie hapert af en toe even. Dat komt doordat `bot.getUpdates()` de NodeMCU kort bezighoudt terwijl hij Telegram checkt. Met een hogere `BOT_MTBS` (bijvoorbeeld 3000) wordt dat minder, maar dan reageert de bot trager.

**✅ Resultaat:**

![Disco-commando in Telegram](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54%20%286%29.jpeg)

![Disco-modus op de strip](images/WhatsApp%20Image%202026-09-23%20at%2020.22.54%20%287%29.jpeg)

---

## De volledige eindcode

Vul je eigen wifigegevens en token in bij de placeholders.

```cpp
/*******************************************************************
    Telegram-bot voor ESP8266 (NodeMCU) die een NeoPixel-strip aanstuurt.
    Gebaseerd op het EchoBot-voorbeeld van Brian Lough:
    https://github.com/witnessmenow/Universal-Arduino-Telegram-Bot
 *******************************************************************/

#include <ESP8266WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <Adafruit_NeoPixel.h>

// Wifi network station credentials
#define WIFI_SSID "JOUW_WIFINAAM"
#define WIFI_PASSWORD "JOUW_WACHTWOORD"
// Telegram BOT Token (Get from Botfather)
#define BOT_TOKEN "JOUW_BOT_TOKEN"

// NeoPixel-strip
#define LED_PIN    D1
#define NUMPIXELS  16
#define HELDERHEID 80

const unsigned long BOT_MTBS = 1000; // mean time between scan messages

X509List cert(TELEGRAM_CERTIFICATE_ROOT);
WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);
unsigned long bot_lasttime; // last time messages' scan has been done

Adafruit_NeoPixel pixels(NUMPIXELS, LED_PIN, NEO_GRB + NEO_KHZ800);
bool discoModus = false;
unsigned long laatsteDiscoStap = 0;

void handleNewMessages(int numNewMessages)
{
  for (int i = 0; i < numNewMessages; i++)
  {
    String text = bot.messages[i].text;
    String chat_id = bot.messages[i].chat_id;

    Serial.println(text);
    text.toLowerCase();
    text.trim();

    if (text == "lights on")
    {
      discoModus = false;
      pixels.fill(pixels.Color(255, 255, 255));
      pixels.show();
      bot.sendMessage(chat_id, "Lights on, captain!", "");
    }
    else if (text == "lights off")
    {
      discoModus = false;
      pixels.clear();
      pixels.show();
      bot.sendMessage(chat_id, "Lights off, captain!", "");
    }
    else if (text == "disco")
    {
      discoModus = true;
      bot.sendMessage(chat_id, "Disco mode activated, captain!", "");
    }
    else
    {
      bot.sendMessage(chat_id, "Unknown command. Try 'lights on', 'lights off' or 'disco'.", "");
    }
  }
}

void setup()
{
  Serial.begin(115200);
  Serial.println();

  pixels.begin();
  pixels.setBrightness(HELDERHEID);
  pixels.clear();
  pixels.show();

  // attempt to connect to Wifi network:
  Serial.print("Connecting to Wifi SSID ");
  Serial.print(WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  secured_client.setTrustAnchors(&cert); // Add root certificate for api.telegram.org

  while (WiFi.status() != WL_CONNECTED)
  {
    Serial.print(".");
    delay(500);
  }
  Serial.print("\nWiFi connected. IP address: ");
  Serial.println(WiFi.localIP());

  Serial.print("Retrieving time: ");
  configTime(0, 0, "pool.ntp.org"); // get UTC time via NTP
  time_t now = time(nullptr);
  while (now < 24 * 3600)
  {
    Serial.print(".");
    delay(100);
    now = time(nullptr);
  }
  Serial.println(now);
}

void loop()
{
  if (millis() - bot_lasttime > BOT_MTBS)
  {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);

    while (numNewMessages)
    {
      Serial.println("got response");
      handleNewMessages(numNewMessages);
      numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    }

    bot_lasttime = millis();
  }

  if (discoModus && millis() - laatsteDiscoStap > 100)
  {
    laatsteDiscoStap = millis();
    for (int i = 0; i < NUMPIXELS; i++)
    {
      pixels.setPixelColor(i, pixels.ColorHSV(random(65536)));
    }
    pixels.show();
  }
}
```

## Wat ik heb geleerd

- **Lees de foutmelding helemaal.** De `note:`-regel bij fout 3 liet precies zien wat de functie verwachtte.
- **De gevaarlijkste fouten geven geen foutmelding.** Fout 2, 6, 7 en 8 compileerden allemaal prima, maar deden iets anders dan ik dacht. `Serial.println` op de juiste plekken helpt om te zien waar het misgaat.
- **Vergelijk met iets wat wel werkt.** Fout 6 vond ik door mijn code te vergelijken met een eerdere opdracht waarin dezelfde strip wel werkte.
- **Hardware bepaalt de code.** Een NeoPixel-strip is geen gewone LED, en een ingebouwde LED werkt omgekeerd (active low).

## Bronnen

- Docent HvA CMD. *Telegram Adafruit ESP8266* (beschrijving ToDo 4). [SharePoint](https://icthva.sharepoint.com/:w:/s/FDMCI_ORG__CMD-Amsterdam/Eb7Jd27yWphMuVFbMHV_9WoBEg5_zqAQilsb6Q3gPSKueg?e=f5PM7l)
- Dekker, K. (2023). *Cheap-Philips-Hue* (voorbeeldmanual). [GitHub](https://github.com/Kvdekker/Cheap-Philips-Hue/blob/main/README.md)
- Lough, B. *Universal-Arduino-Telegram-Bot* (library en EchoBot-voorbeeld). [GitHub](https://github.com/witnessmenow/Universal-Arduino-Telegram-Bot)
- Blanchon, B. *ArduinoJson*. [arduinojson.org](https://arduinojson.org)
- Adafruit. *Adafruit NeoPixel Library*. [GitHub](https://github.com/adafruit/Adafruit_NeoPixel)
- Adafruit. *Adafruit NeoPixel Überguide*. [learn.adafruit.com](https://learn.adafruit.com/adafruit-neopixel-uberguide)
- Telegram. *Telegram Bot API*. [core.telegram.org](https://core.telegram.org/bots/api)
- ESP8266 Arduino Core. *Documentatie (WiFi)*. [arduino-esp8266.readthedocs.io](https://arduino-esp8266.readthedocs.io)
- Arduino. *Arduino IDE*. [arduino.cc](https://www.arduino.cc/en/software)
- Anthropic. *Claude* (AI-assistent), gebruikt als hulp bij het debuggen en het uitleggen van foutmeldingen.
- Eigen code: weerstation-opdracht (IoT), gebruikt als werkende referentie voor de NeoPixel-strip.
