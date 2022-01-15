## A few words in Finnish
Mikäli olet kiinnostunut saamaan lisätietoa tästä voit lähettää tekijälle sähköpostia olli@rinne.fi. Keskustelua voisi tietysti käydä myös FB:n Aurinkoenergia-ryhmässä, josta ajatus tämän julkaisusta tuli. Voit myös halutessassi avata issuen suomeksi. Käytännön syistä dokumentaatio on kuitenkin ainakin toistaiseksi vain englanniksi.

Ohjelma getfcstandprices.py hakee paikallisen aurinkoenergiaennusteen ja Nordpoolin day-ahead SPOT-tuntihinnat (vaatii ilmaisen API-avaimen). SPOT-hintoja ei vielä käytetä ohjauksessa, mutta ominaisuus on melko helppo lisätä ja tekijä mielellään on mukana tässä. Käytännössä hintaohjauksen voisi tehdä lisäämällä esimerkiksi ehtokriteereihin (laajentamalla funtiota check_conditions ) totuusarvomuuttujat  (boolean) spotlow5 (true, jos kuluva tunti kuuluu 5 halvimman tunnin joukkoon), jne.spotlow10, spothigh10, spothigh5 . Ehtoihin voisi tietysti myös lisätä myös absoluuttiseen hintaan viittaavat spotpricebelow and spotpriceabove-attribuutit, jolloin esim. "spotpricebelow" : 3.0 on voimassa kun SPOT-hinta on alle 3 c/kWh. jne .

## Idea
TO BE UPDATED...
Powerguru manages electric loads (especially 1 or 3 phase water heaters). It can heat up the boilers when then electricity is cheap, for example when you have excess solar power or nightime. It can also optimize water heating using solar energy forecast (http://www.bcdcenergia.fi/ for forecast in Finland). Current version can read RS485/Modbus enables electric meters and DS18B20 temperatare sensors. It can also fetch Nordpool day-ahead spot prices. 

It calculates target temperatures of the heaters once in a minute and switches on/off the heater resistors to reach current target value. Dynamic target values (in Celcius) depends on current "conditions", which are enabled if all the criterias for the condition match.   Powerguru is tested with Raspberry Pi (2)

## Data architechture
[Telegraf](https://github.com/influxdata/telegraf) is a plugin-driven server agent for collecting & reporting metrics. Telegraf gets metrics from sensors and other data sources through input plugins and forwards it to Powerguru and influxDB analytics database (optional) through output plugins. In addition to standard Telegraf plugins, custom Powerguru Telegraf input plugins (see colors in the diagram) are used. 

**Powerguru** is a multithreaded Python-program running as a Linux service, see details below. Powerguru has a inbuild web-server (aiohttp) for communication with Telegraf and dashboard. Currently external devices (e.g. boilers) are controlled with Raspberry PI GPIO driven switches, but addional device interfaces, throught e.g. http API:s can be added.

[InfluxDB](https://www.influxdata.com/) is an open-source time series database . [Grafana](https://grafana.com/) is an open source analytics and interactive visualization web application. Analytics of collected metrics and data is optional. A cloud based service, e.g. [InfluxDB Cloud](https://www.influxdata.com/products/influxdb-cloud/) is probobly easiest to start with. If you like to host InfluxDB locally, use other storage media than a micro SD card, which is not designed for frequent writes. 

![Data flow diagram](https://github.com/Olli69/powerguru/blob/main/docs/img/Powerguru%20data%20diagram.drawio.png?raw=true)

### Telegraf - Powerguru communication
Telegraf [outputs.http plugin](https://github.com/influxdata/telegraf/blob/release-1.21/plugins/outputs/http/README.md) sends buffered metrics updates to Powerguru http interface. Powerguru calculation data series are updated to an optional InfluxDB database via Telegraf proxy service [inputs.influxdb_v2_listener](https://github.com/influxdata/telegraf/blob/release-1.21/plugins/inputs/http_listener_v2/README.md).

### Modbus energy meter
Metrics from a Modbus enabled energy meter is fetched with [Telegraf Modbus Input Plugin](https://github.com/influxdata/telegraf/blob/master/plugins/inputs/modbus/README.md). The plugin support TCP and serial line configuration. Currently Carlo Gavazzi EM340 RS meter is supported by the [Telegraf config file](settings/telegraf-powerguru.conf).

### 1-wire temperatore sensors
Temperature data from 1-wire temperature sensor DS18B20 is supported. For plugin code see [onew_telegraf_pl.py](onew_telegraf_pl.py) .

### EntsoE
Day-ahead spot prices are fetched from [EntsoE transparency platform](https://transparency.entsoe.eu/). Next day NordPool prices are available in afternoon. For plugin code see [entsoe_telegraf_pl.py](entsoe_telegraf_pl.py) .

### BCDC Energia
BCDC Energia gives day-ahead solar-power forecast for specified locations in Finland. Data is fetched several times a day. For plugin code see [bcdc_telegraf_pl.py](bcdc_telegraf_pl.py) .

### Solar Inverters
Production data can be updated from solar (PV) inverters with a HTTP-api (e.g. Fronius Solar Api). [Telegraf HTTP input plugin](https://github.com/influxdata/telegraf/blob/release-1.21/plugins/inputs/http/README.md)

### Dashboard
<img align="right" width="100" height="100" src="https://github.com/Olli69/powerguru/blob/main/docs/img/powerguru-dashboard.png?raw=true">
Dashboard is a tiny web service showing current state of Powerguru service. You can see:
- Incoming/outgoing energy
- Currently enables conditions
- Status of the channels
- Current values of variables, which are used to control statuses and channel targets
- Update status of different data from Telegraf to Powerguru



## Concept
TO BE UPDATED...
Main program powerguru.py runs function doWork and does following once in a minute (parameter READ_INTERVAL) in 
1. reads ModBus capable electic meter and temperature sensors .
2. Insert new values to InfluxDB
3. Checks which conditions are valid in that moment. Multiple conditions can be valid at the same time,
4. Searches targets of each actuator in order. E.g. target {"condition" : "sun", "sensor" : "b2out", "valueabove": 80} means that if sun (net sales) condition is true, sensor b2out target value if 80 C. If condition "sun" is true but sensor "b2out" temperature is below 80 then the system tries to switch on more lines (if possible). If temperature is above 80, it can switch of the lines (of that actuator).
5. Does actual switching with lineResource.setLoad . This function can also switch of the lines if there is too much load on a phase.
### Conditions
At any time multiple conditions can be effective. Conditions are enabled based on one or more criterias, which should be fulfilled:
- current time, criteria defined with _starttime_ (e.g. "04:00:00") and _endtime_ (e.g. "07:00:00)
- current date, defined with parameters _dayfirst_ and _daylast_  -  format MM-DD,  e.g. "02-15" is February 15
- solar forecast, attributes _solar12above_ and  _solar12below_ are defined. _"solar12above" : 5.0_ means that expected cumulative solar power within next 12 h should be 5kWh or more (with 1 kWp panels)
- in the future there could be criterias for price-based selection, e.g. _spotpricebelow_
Condition parameter are defined in settings.py file.
### Actuators
Currently only (1 or 3 line) boilers/heaters are supported. Actuator defines GPIOs of all phases (1 or 3) and target values (temperatures) in different conditions. Targets are tested in order and first matching target is used. Actuators are defined in settings.py file.
### Sensors
Currently only DS18B20 1-wire temperature sensors are supported. Sensors are identified by id and  defined in settings.py file.

## Installation
TO BE UPDATED...

### Files
TO BE UPDATED...
* powerguru.py - main program file. Starts from command line:  python3 powerguru.py or run as systemd service (see powerguru.service file)
* bcdc_telegraf_pl.py, entsoe_telegraf_pl.py, onew_telegraf_pl.py - custom Powerguru Telegraf input plugins
* powerguru.service - systemd service template, edit and install if you like to run powerguru as daemon
* README.md - this file, will be completed 
* setting/channels.json  
* setting/conditions.json  
* setting/powerguru.json  
* setting/sensors.json  
* setting/telegraf-powerguru.conf

 

### Required Python components

TO BE UPDATED...
todo: one line, updagrade
sudo apt-get install libatlas-base-dev python3-pip

sudo -H pip3 install pytz  python-dateutil twisted pymodbus influxdb entsoe-py

 
.... all the other required libraries,

### Wiring
GPIOs are defined in actuators in the file settings.py. 3-phase heater uses 3 GPIOs if you want to control lines individually. In the pilot installation GPIOs draw SSR switches, which are connected to AC relays. (Maybe a LN2003 drawing a DC connector could be simpler.)

#### Wiring to a boiler, 3 phase boler has 3 of these
GPIO numbers are defined in _actuators_ list in file _settings.py_ .

    RPi GPIO  -------- 
                      SSR switch -------- AC switch  (leave to an electrician!)-------   Boiler
    RPi GND   -------- 
    
#### Electricity meter reading with Modbus
OR USE USB DONGLE... will be updated
Use raspi-config to disable serial console and enable serial port.

     RPi GPIO14 (TXD)  --------   RX               ---D+----
     RPi GPIO15 (RXD)  --------   TX    MAX3485    ---D1----    Carlo Gavazzi EM340 RS(leave to an electrician!)
     RPi 3.3V.         --------   VCC              ---GND---
     RPi GND           --------   GND
     
 
     
    
DS18B20 sensors are wired and terminated (see one-wire wiring) and how to enable one-wire https://pinout.xyz/pinout/1_wire.  Each sensor is identified by unique id and you can get recognized sensor id:s with command: `ls /sys/bus/w1/devices/`. In the beginning  all available sensors are searched in function find_thermometers and mapped to sensor codes defined in _sensors_ list defined in _settings.py_ .

    RPi GPIO 4 ----------------          --------          --- ... -----
                    |
                 4.7k ohm pull-up
                    |
    RPi 3V3    ----------------  DS18B20 --------  DS18B20 --- ... -----
    
    RPi GND    ----------------          --------          --- ... -----

Sensors should be bind to warmest part of the pipeline (outside), so that it get as hot as possible (may silicon paste and insulation outside could help). Anyway keep in minds that sensor values will be lower than real water temperature.  See mounting example https://www.openheating.org/doc/faschingbauer/thermometers.html 







