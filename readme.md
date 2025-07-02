# Práca s protokolmi HTTP a MQTT v jazyku MicroPython

![Raspberry Pi Pico W](images/banner.jpg)

Doska _Raspberry Pi Pico_ patrí do rodiny zariadení vyrobených nadáciou _Raspberry Pi Foundation_. Nejedná sa však o minipočítač, ako je to v prípade minipočítačov _Raspberry Pi_, ale jedná sa o dosku osadenú mikrokontrolérom _RP2040_. Vďaka cene, svojim vlastnostiam, komunite a podpore nadácie [Raspberry Pi Foundation](https://www.raspberrypi.org/), je doska _Raspberry Pi Pico_ ideálnou pomôckou na výučbu programovania mikrokontrolérov. A počas tejto tvorivej dielne si predstavíme základy práce s touto doskou v jazyku _MicroPython_. Vytvoríme jednoduché riešenie, dosku pripojíme do internetu, potom pomocou HTTP protokolu stiahneme informácie o počasí, aby sme ich následne uložili do _Adafruit IO_ pomocou protokolu HTTP aj MQTT.

**Odporúčaný čas:** 180 minút


## Ciele

1. Naučiť sa nainštalovať firmvér s jazykom _MicroPython_.
1. Naučiť sa základy práce v REPL režime jazyka _MicroPython_.
1. Naučiť sa pripojiť dosku _Raspberry Pi Pico W_ do internetu pomocou zabudovaného _WiFi_ modulu.
1. Naučiť sa publikovať údaje z mikrokontroléra pomocou komunikačného protokolu _MQTT_.
1. Naučiť sa základy používania modulu `urequests` pre prácu s HTTP protokolom.


## Čo budeme robiť

Na tomto workshope si vyskúšame základy práce s mikrokontrolérom _Raspberry Pi Pico W_, kde si ukážeme práve to, ako je možné komunikovať pomocou HTTP a MQTT protokolu. Ukážeme si, ako do neho nainštalovať firmvér, ako sa priprojiť na WiFi sieť, stiahnuť dáta o aktuálnom počasí, a pretransformujeme ich do formy, ktorú potrebuje služba [Adafruit IO].

Vytvoríme vlastne jednoduché ETL, ktoré bude vyzerať takto:

```raw
[ extract_data ] --> [ transform_data ] --> [ load_data ]
```

Každú jednu úlohu pritom implementujeme vo forme samostatnej Python funkcie.


## Naivné riešenie

Vytvárať budeme veľmi naivné riešenie, kde sa nebudeme venovať mnohým kontrolám, ktoré by sme pri reálnom nasadení riešili, ako napr. HTTP status odpovedí, výpadok spojenia a pod. Niekoľko odporúčaní sa tu ale objaví.


## Niekoľko tipov pre prácu s mikrokontrolérom

* Pico na doske neobsahuje tlačidlo `RESET`. Reštartovať ho je však jednoduché odpojením a opätovným pripojením USB kábla. **Vyhnite sa však jeho odpájaniu a pripájaniu na strane mikrokontroléra!** Vyhnete sa tak možnosti odtrhnutia tohto konektoru a tým pádom aj jeho celkovému poškodeniu. Miesto toho odpájajte USB kábel na strane počítača.

* V prípade, že chcete mikrokontrolér reštartovať softvérovo, môžete tak urobiť príkazom:

  ```python
  from machine import reset; reset()
  ```

* Softvérový reset zariadenia sa dá vykonať ešte jednoduchšie - stlačením klávesovej skratky `CTRL+D`


## Krok 0. Čo budeme potrebovať?

Ešte predtým, ako začneme, si pripravte nasledovné:

* Na svoj počítač nainštalujte jednoduchý editor kódu [Thonny] a v jeho menu `Zobraziť` zaškrtnite voľby `Shell` a `Súbory`.

* Ak pracujete v _OS Linux_, tak sa uistite, že váš používateľ má práva na prístup k zariadeniu (obyčajne je to skupina `dialout`). Ak to tak nie je, pridajte používateľa do uvedenej skupiny napr. týmto príkazom:

    ```bash
    $ sudo usermod -aG dialout USERNAME
    ```

* Ak pracujete v _OS Windows_, uistite sa, že máte správne ovládače, a že systém mikrokontrolér rozpozná.

* Vytvorte si bezplatný účet v službe [Open Weather].

* Vytvorte si bezplatný účet v službe [Adafruit IO].


## Krok 1. Nahratie firmvéru do mikrokontroléra

Ak chceme mikrokontrolér programovať v jazyku _MicroPython_, potrebujeme najprv do neho nahrať firmvér s týmto jazykom. S tým nám pomôže editor _Thonny_.

Pre nahratie firmvéru postupujte podľa tohto návodu:

1. Spustite editor _Thonny_.
2. Ak máte dosku _Raspberry Pi Pico_ pripojenú k počítaču, tak ju odpojte. Na doske stlačte a držte tlačidlo `BOOTSEL` a dosku pripojte k počítaču.
3. Po pripojení dosky tlačidlo pustite. Váš operačný systém môže zobraziť notifikáciu o tom, že k vášmu počítaču bol pripojený USB disk s názvom `RPI-RP2`.
4. V pravom dolnom rohu editora _Thonny_ kliknite na názov používaného interpretera jazyka _Python_  a zo zoznamu možností vyberte položku `Inštalovať MicroPython...`.

   ![Editor Thony: Výber inštalácie jazyka MicroPython](images/thonny-corner.png)
5. V dialógovom okne `Inštalovať MicroPython` vyberte umiestenie pripojenej dosky Raspberry Pi Pico, vyberte variant dosky a verziu jazyka MicroPython, ktorú chcete nainštalovať.

   ![Editor Thonny: Dialóg pre inštaláciu jazyka MicroPython](images/thonny-install.micropython.png)
   
6. Kliknite na tlačidlo `Inštalovať`, čím sa spustí inštalácia.
7. Po skončení inštalácie zatvorte dialógové okno a v pravom dolnom rohu editora vyberte zo zoznamu vaše zariadenie s jazykom _MicroPython_ (napr. `MicroPython (Raspberry Pi Pico)`) .

Ak ste postupovali správne, v príkazovom riadku editora _Thonny_ by ste mali vidieť prompt jazyka _MicroPython_, napr.:

```
MicroPython v1.21.0 on 2023-10-06; Raspberry Pi Pico W with RP2040
Type "help()" for more information.
>>>
```


## Krok 2. Režim REPL a Hello world!

Jediný program, ktorý je v mikrokontroléri nahratý a spustený, je interpreter jazyka _MicroPython_. Komunikovať s ním je možné pomocou sériovej linky, kde uvidíme jeho rozhranie v režime REPL. To môžeme otestovať vypísaním textu `Hello world!` pomocou funkcie `print()`:

```python
>>> print('Hello world!')
Hello world!
>>>
```

Podobne, ako v prípade štandardného jazyka _Python_ môžeme režim REPL využiť na ladenie a experimentovanie.


## Krok 3. Pripojenie k internetu

Ako základ pre pripojenie do internetu použijeme funkciu z [dokumentácie jazyka MicroPython](https://docs.micropython.org/en/latest/esp32/quickref.html#networking), ktorá vyzerá takto:

```python
def do_connect():
    import network
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print('connecting to network...')
        wlan.connect('ssid', 'key')
        while not wlan.isconnected():
            pass
    print('network config:', wlan.ifconfig())
```

Mierne ju upravíme, aby jej parametrom bolo SSID pre pripojenie do siete a rovnako tak aj heslo:

```python
def do_connect(ssid, password):
    import network
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print(f'>> Connecting to network "{ssid}"...')
        wlan.connect(ssid, password)
        while not wlan.isconnected():
            pass
    print('>> Network config:', wlan.ifconfig())
```

Funkciu môžeme rovno otestovať v _REPL_ režime. Ak sa napríklad chceme pripojiť do siete `labak` s heslom `jahodka`, tak to urobíme takto:

```python
do_connect('labak', 'jahodka')
```

### Odporúčanie

Po vykonaní práce sa nezabudnite od siete odpojiť! To je dôležité kvôli zníženiu spotreby a uvoľneniu prostriedkov aj pre iných.

Odpojiť sa je možné pomocou dvoch metód nad sieťovým rozhraním:

1. `.disconnect()` - odpojí sa od siete, ale WiFi modul zostane aktívny a tým pádom spotreba bude vyššia
2. `.deinit()` - okrem odpojenia aj vypne WiFi modul, čím dôjde aj k zníženiu spotreby zariadenia


## Krok x. Stiahnutie aktuálneho počasia

Cieľom prvej úlohy v našom ETL je stiahnutie informácií o aktuálnom počasí. Vytvoríme preto funkciu `extract_data()`, v ktorej túto úlohu implementujeme.

Na získanie informácií o aktuálnom počasí budeme používať službu [Open Weather]. Služba poskytuje REST API, pomocou ktorého je rozlične sa dopytovať a získavať či už aktuálne alebo historické dáta o počasí.

Pre použitie dát zo služby je potrebné mať účet. Aktuálne počasie budeme vedieť získať aj pomocou bezplatného účtu. Na to použijeme modul `requests`, ktorý je mikro implementáciou dobre známeho a veľmi populárneho _Python_ modulu s rovnomenným názvom [`requests`](https://requests.readthedocs.io/en/latest/)


### Formát URL adresy

Obecne sa URL adresa skladá z niekoľkých častí:

* schéme alebo protokol
* host
* port
* cesta
* parametre požiadavky

Jednotlivé jej časti je možné vidieť na nasledovnom obrázku:

![Formát URL adresy](images/url.format.explained.png)


### HTTP API

Dokumentácia pre HTTP požiadavky na získanie aktuálneho počasia sa nachádza na [tejto stránke](https://openweathermap.org/current).

Ak by sme napríklad chceli získať informácie o aktuálnom počasí v Žiline v metrickej sústave, jej URL adresa bude vyzerať takto:

```
https://api.openweathermap.org/data/2.5/weather?q=zilina&units=metric&appid=9e547051a2a00f2bf3e17a160063002d
```

Samotnú HTTP požiadavku si môžeme overiť okrem prehliadača aj z príkazového riadku pomocou HTTP klientov, ako je napríklad `httpie` takto:

```bash
$ http https://api.openweathermap.org/data/2.5/weather \
  q==zilina \
  units==metric \
  appid==9e547051a2a00f2bf3e17a160063002d
```

### HTTP požiadavka pomocou modulu `requests`

V jazyku MicroPython si takýto dopyt môžeme vyskúšať priamo v REPL režime takto:

```python
>>> import requests
>>> response = requests.get('https://api.openweathermap.org/data/2.5/weather?q=zilina&units=metric&appid=9e547051a2a00f2bf3e17a160063002d')
```

Výsledkom bude objekt typu `Response`, ktorý má niekoľko metód. Keďže vieme, že odpoveď bude vo formáte JSON, môžeme nad objektom odpovede rovno zavolať metódu `.json()`:

```python
>>> response.json()
```


### Riešenie

Vytvoríme teda funkciu `extract_data()`, ktorá bude mať 3 parametre:

* `query` - požiadavku alebo mesto, o ktorom chceme získať informácie
* `units` - v akých jednotkách bude výsledná hodnota (`metric | imperial | standard`)
* `appid` - aplikačný kľúč

Funkcia vráti slovník s načítanými údajmi.

Riešenie môže vyzerať napríklad takto:

```python
import requests

def extract_data(query: str, units: str, appid: str) -> dict:
    url = (
        'https://api.openweathermap.org/data/2.5/weather?'
        f'q={query}&units={units}&appid={appid}'
    )

    response = requests.get(url)
    data = response.json()
    response.close()

    return data
```


## Krok x. Spracovanie prijatých údajov

Našim cieľom je dostať získané dáta o počasí do služby [Adafruit IO]. Najprv sa však pozrieme na to, v akom formáte a v akom tvare služba od nás tieto dáta prijme a pred samotným odoslaním získané dáta do tohto tvaru transformujeme.

[![Adafruit IO](images/logo-adafruitio.png)](https://io.adafruit.com/bletvask)


### Adafruit IO kanály

Ešte skôr, ako začneme, potrebujeme sa zoznámiť s konceptov tzv. _kanálov_ (z angl. _feeds_(. V [Adafruit IO] sú kanály základným stavebným prvkom, pomocou ktorého sa ukladajú a organizujú dáta. Každý kanál je ako databázová tabuľka alebo kontajner, ktorý uchováva chronologický záznam hodnôt.

O kanáloch platí:

* Kanál je miesto, kam môžeš odosielať údaje (napr. teplota, vlhkosť, stav tlačidla atď.).

* Do kanála je možné údaje nielen zapisovať ale aj čítať.

* Používa sa na vizualizáciu údajov cez dashboardy (grafy, gauge, tlačidlá, prepínače).

* Každý kanál má svoje jedinečné meno a môže byť verejný alebo súkromný.

* Viacero kanálov môže byť organizovaných do jednej skupiny.

Pre naše potreby si teda vytvoríme skupinu `ossconf` a v nej si vytvoríme kanály pre merané veličiny `temp`, `humidity` a `pressure`.


### Formát odosielaných dát

Dáta, ktoré budeme odosielať, musia byť v JSON formáte a budú obsahovať tieto kľúče:

* `lat` - zemepisná šírka
* `lon` - zemepisná dĺžka
* `value` - nameraná hodnota
* `created_at` - čas merania v ISO formáte 

Takže JSON obsahujúci meranie v Žiline môže vyzerať takto:

```json
{
    "lat": 49.2239,
    "lon": 18.7394,
    "value": 9.79,
    "created_at": "2025-07-02T06:23:00+02:00"
  }
```


### Spracovanie údajov

Ešte predtým, ako sa však dostaneme ku samotnému odoslaniu dát do služby [Adafruit IO] v jazyku _MicroPython_, vytvoríme si dve pomocné funkcie.

Prvá funkcia zabezpečí jednoduchú konverziu časovej značky do ISO8601 formátu času.

```python
def to_iso8601(ts: int) -> str:
    dt = time.gmtime(ts)
    return f'{dt[0]:04}-{dt[1]:02}-{dt[2]:02}T{dt[3]:02}:{dt[4]:02}:{dt[5]:02}Z'
```

Druhá funkcia bude reprezentovať samotnú úlohu nášho ETL pre transformáiu dát. Jej úlohou bude extrahovať len tie dáta, o ktoré máme záujem. To môže vyzerať napríklad takto:

```python
def transform_data(data: dict) -> dict:
    return {
        'dt'       : to_iso8601(data['dt']),
        'lat'      : data['coord']['lat'],
        'lon'      : data['coord']['lon'],
        'metrics'  : {
            'temp'     : data['main']['temp'],
            'humidity' : data['main']['humidity'],
            'pressure' : data['main']['pressure']
        }
    }
```


## Krok x. Odoslanie údajov do Adafruit IO cez protokol HTTP

### Odoslanie metriky cez protokol HTTP

Do každého kanálu môžeme odosielať údaje pomocou HTTP metódy `POST`. Každý kanál má pre tento účel k dispozícii konkrétnu URL adresu, ktorú získame po kliknutí na položku `Feed Info` v pravom bočnom paneli príslušného kanála. URL adresu nájdete pod kľúčom `API`.

Okrem toho sa však pri každej jednej požiadavke potrebujeme aj autentifikovať pomocou používateľského mena a kľúča. Tieto hodnoty získame po kliknutí na ikonku kľúča, kde tieto hodnoty získame, prípadne ich môžeme nechať znovu vytvoriť.

Overiť zápis do príslušného kanála z príkazového riadku môžeme nasledovne:

```bash
$ http post \
  https://io.adafruit.com/api/v2/{user}/feeds/{group}.{feed}/data \
  X-AIO-Key:{aio_key} \
  datum:='{
    "lat": 49.2239,
    "lon": 18.7394,
    "value": 9.79,
    "created_at": "2025-07-02T06:23:00+02:00"
  }'
```


### Odoslanie metriky v jazyku _MicroPython_

V jazyku _MicroPython_ v _REPL_ režime to môžeme urobiť takto:

```python
>>> import requests

>>> user = 'jano'
>>> group = 'ossconf'
>>> feed = 'temp'
>>> aio_key = 'secret.password'

>>> url = f'https://io.adafruit.com/api/v2/{aio_username}/feeds/{group}.{feed}/data'

>>> payload = {
    "lat": 49.2239,
    "lon": 18.7394,
    "value": 9.79,
    "created_at": "2025-07-02T06:23:00+02:00"
}

>>> headers = {'X-AIO-Key': aio_key}
>>> response = requests.post(url, headers=headers, json=payload)
```


### Riešenie

Výsledná funkcia môže vyzerať napríklad takto:

```python
def load_data(aio_username: str, aio_key: str, group: str, data: dict):
    headers = {'X-AIO-Key': aio_key}

    for feed in data['metrics']:
        url = f'https://io.adafruit.com/api/v2/{aio_username}/feeds/{group}.{feed}/data'

        payload = {
            'value': data['metrics'][feed],
            'lat': data['lat'],
            'lon': data['lon'],
            'created_at': data['dt']
        }

        response = requests.post(url, headers=headers, json=payload)
        response.close()
```


## Krok x. Odoslanie údajov do Adafruit IO cez protokol MQTT

Do kanálov je možné odosielať dáta aj pomocou protokolu MQTT. A presne to urobíme v tomto kroku.


### Nainštalovanie balíka pre MQTT

Podpora protokolu MQTT vo firmvéri jazyka _MicroPython_ nie je žiadna. Aby teda náš mikrokontrolér vedel komunikovať aj pomocou tohto protokolu, musíme do neho doinštalovať príslušný modul. 

To vieme urobiť priamo z prostredia editora [Thonny] pomocou modulu [pipkin](https://github.com/aivarannamaa/pipkin). Problém však je, že od istej verzie jazyka _Python_ už nepracuje správne.

Takže modul pre protokol MQTT nainštalujeme v _REPL_ režime manuálne pomocou modulu `mip` nasledovne:

```python
>>> import mip
>>> mip.install('umqtt.simple')
```

Po nainštalovaní tento modul uvidíte aj v súborovom systéme v priečinku `lib/`.


### Pripojenie k MQTT brokeru

MQTT broker pre službu [Adafruit IO] sa nachádza na adrese `io.adafruit.com` a komunikuje na štandardnom porte `1883`. Pri pripájaní použijete svoje používateľské meno a vygenerovaný kľúč, ku ktorému sa dostanete cez ikonu kľúčika (používali sme ho ako premennú `aio_key`).

Tému pre príslušný kanál zistíte opať v bočnom paneli kanála v položke `Feed Info`.


### Odoslanie metriky cez protokol MQTT

To, či je všetko v poriadku, je možné otestovať z príkazového riadku napr. pomocou `mosquitto_pub` nástroja takto:

```bash
$ mosquitto_pub -h io.adafruit.com -u {user} -P {key} \
  -t {user}/feeds/{group}.{feed} \
  -m '{
    "lat": 49.2239,
    "lon": 18.7394,
    "value": 9.79,
    "created_at": "2025-07-02T06:23:00+02:00"
  }'
```


### Odoslanie skupiny metrík cez protokol MQTT

Ak by ste chceli na jeden šup odoslať viacero metrík, je to možné dosiahnuť týmto spôsobom:

```bash
$ mosquitto_pub -h io.adafruit.com -u {user} -P {key} \
  -t {user}/groups/{group} \
  -m '{
    "feeds": {
      "pressure": 1016,
      "humidity": 94,
      "temp": 7.54
    },
    "location": {
      "lat": 48.6667,
      "lon": 21.3333
    }
  }'
```


### Odoslanie údajov v jazyku MicroPython

Ak chceme odoslať meranie v jazyku _MicroPython_, môžeme si to vyskúšať v _REPL_ režime pomocou týchto príkazov:

```python
>>> user = 'jano'
>>> group = 'ossconf'
>>> feed = 'temp'
>>> aio_key = 'secret.password'
>>> topic = f'{aio_username}/groups/{group}'

>>> payload = {
    "lat": 49.2239,
    "lon": 18.7394,
    "value": 9.79,
    "created_at": "2025-07-02T06:23:00+02:00"
}

>>> from umqtt.simple import MQTTClient

>>> client = MQTTClient(unique_id(), 'io.adafruit.com', 1883, 
                        aio_username, aio_key)
>>> client.connect()
>>> client.publish(topic, json.dumps(payload))
>>> client.disconnect()
```


### Výsledné riešenie

Výsledná funkcia môže vyzerať napríklad takto:

```python
from umqtt.simple import MQTTClient

def load_data_mqtt(aio_username: str, aio_key: str, group: str, data: dict):
    topic = f'{aio_username}/groups/{group}'

    payload = {
        "feeds": {
            "temp": data['metrics']['temp'],
            "humidity": data['metrics']['humidity'],
            "pressure": data['metrics']['pressure'],
        },
        "location": {
            'lat': data['lat'],
            'lon': data['lon'],
        }
    }

    client = MQTTClient(unique_id(), 'io.adafruit.com', 1883, aio_username, aio_key)
    client.connect()
    client.publish(topic, json.dumps(payload))
    client.disconnect()
```


## Krok x. Pravidelné spúšťanie

### ...v nekonečnej slučke

```python
if __name__ == '__main__':
    while True:
        main()

        print(f'>> going to sleep for {INTERVAL} minutes')
        time.sleep(INTERVAL * 60)
```

### ...hlbokým uspaním

```python
if __name__ == '__main__':
    main()

    print(f'>> going to sleep for {INTERVAL} minutes')
    machine.deepsleep(INTERVAL * 60 * 1000)
```


## Bonus: Dekorátor osvietenia

```python
from machine import Pin

def enlight(func):
    def wrapper(*args, **kwargs):
        led = Pin('LED', Pin.OUT)
        led.on()
        func(*args, **kwargs)
        led.off()
    return wrapper
```





## Krok x. Aktualizácia času

Počas pripájania k internetu vieme rovno synchronizovať aj čas pomocou NTP protokolu. Aktualizujeme a upravíme funkciu `do_connect()` nasledovne:

```python
def do_connect(ssid, password):
    import network, ntptime, machine
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print('>> Connecting to network...')
        wlan.connect(ssid, password)
        while not wlan.isconnected():
            pass
    print('>> Network config:', wlan.ifconfig())

    # set time and date with NTP
    print('>> Synchronizing time...')
    ntptime.settime()
    rtc = machine.RTC()
    now = rtc.datetime()
    print(f'>> Current time: {now[0]}-{now[1]:02}-{now[2]:02}T{now[4]:02}:{now[5]:02}:{now[6]:02}Z')
```

**Poznámka:** Pri pripojení pomocou editora ako Thonny alebo nástroja `rshell`, dôjde k synchronizácii času automagicky. Počas reálnej prevádzky synchronizáciu musíme zabezpečiť sami.

## Krok x. Inštalácia doplnkových modulov

Pred verziou 1.20 sa používal modul `upip`, ktorý predstavoval MicroPython verziu nástroja `pip` pre inštaláciu balíkov. Od verzie 1.20 je však súčasťou firmvéru s MicroPython-om balík s názvom `mip`. Ten neinštaluje balíky z [PyPi](https://pypi.org), ale z repozitára [micropython-lib](https://github.com/micropython/micropython-lib):

Nainštalovať nový modul môžeme priamo pomocou REPL. Najprv importneme modul `mip`:

```python
>>> import mip
```

A následne nainštalujeme príslušný balík pomocou:

```python
>>> mip.install('umqtt.simple')
```

Inštalované moduly sa inštalujú do priečinku `/lib/` v mikrokontroléri.




## Ďalšie zdroje

* [Raspberry Pi Pico and Pico W](https://www.raspberrypi.com/documentation/microcontrollers/raspberry-pi-pico.html) - technická dokumentácia pre rodinu mikrokontrolérov _Raspberry Pi Pico_
* [Package management](https://docs.micropython.org/en/latest/reference/packages.html?highlight=mip) - dokumentácia ku modulu `mip` pre správu balíčkov na mikrokontroléri s jazykom MicroPython
* [Propojka z Grove konektoru na 4 pin dupont samice](https://rpishop.cz/propojky/2167-propojka-z-grove-konektoru-na-4-pin-dupont-samice-5-kusu-v-baleni.html)
* https://wokwi.com/
* [Raspberry Pi Pico W 101](https://namakanyden.sk/webinars/2024.01-rpi.pi.pico.w.101.html) - záznam z webinára


## Licencia

Uvedené dielo podlieha licencii [Creative Commons BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.cs).







[Adafruit IO]: https://io.adafruit.com/
[Open Weather]: https://openweathermap.org/
[Thonny]: https://thonny.org/
