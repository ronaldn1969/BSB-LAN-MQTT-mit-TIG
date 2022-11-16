## Was ist BSB-LAN

"BSB-LPB-LAN" ist ein gemeinschaftliches Hard- und Softwareprojekt, welches ursprünglich zum Ziel hatte, mittels PC/Laptop/Tablet/Smartphone Zugriff auf die Steuerungen bzw. Regler von verschiedenen Wärmeerzeugern (Öl- und Gasheizungen, Wärmepumpen, Solarthermie etc.) bestimmter Hersteller (bspw. Brötje und Elco) zu erlangen. Im weiteren Verlauf wäre es dann wünschenswert, Daten auszulesen, sie weiter zu verarbeiten (z.B. loggen und grafisch darstellen) oder gar Einfluss auf die Steuerung/Regelung nehmen zu können und das System in bestehende SmartHome-Systeme einzubinden.

Weitere Inflormationen gibt es hier: https://github.com/1coderookie/BSB-LPB-LAN

## Meine Installation

Da ich schon meine FritzBox, Raspberry Pi, Shellies und Govee Thermometer schon über Mosquitto, Telegraf, InfluxDB und Grafana (als Docker Container) monitore, war es für mich klar, den BSB-LAN Adapter hier ebenfalls zu integrieren. 

Ausgang meiner TIG Installation ist das Repo von Phil Hawthorne. Danke für die Vorarbeit. Ich habe einen Teile der Dateien (Dockerfile und INIs) auf meine Bedürfnisse angepasst. Da ich Chronograf nicht nutze, habe ich es aus der Installation entfernt und Telegraf hinzugefügt. Des Weiteren habe ich die Versionen aktualisiert.

Anmerkung von Phil: This is a Docker image based on the awesome [Docker Image with Telegraf (StatsD), InfluxDB and Grafana](https://github.com/samuelebistoletti/docker-statsd-influxdb-grafana) from [Samuele Bistoletti](https://github.com/samuelebistoletti).


### Anpassungen in der telegraf.conf, damit die MQTT Nachrichten vom BSB-LAN in eine InfluxDB geschrieben werden

Voraussetzung ist, dass Ihr Telegraf, InfluxDB und Mosquitto auf Eurem System laufen habt. Wenn nicht, könnt Ihr weiter unten entnehmen, wie Ihr Euch einen TIG Container erstellt und Mosquitto installiert. Es ist kein Zwang, Docker zu verwenden.

Des Weiteren müsst Ihr in den BSB-LAN Einstellungen, bei MQTT Verwenden, Rich JSON auswählen. Dann werden die Daten in folgender Struktur geliefert:

```
BSB-LAN/json {"BSB-LAN":{"id":8700,"name":"Aussentemperatur","value": "12.8","desc": "","unit": "°C","error": 0}}
```

Bitte auch beachten, dass das Loginterval im Standard auf 3600 Sekunden steht. Ich habe es für mich auf 60 Sekunden reduziert.

**Anmerkung:** Ich nutze InfluxDB v1.8.10. Es kann sein, dass die Anpassungen für InfluxDB v2.x nicht passen.

```
# Diese Zeilen bitte Eurer telegraf.conf hinzufügen 

# Daten aus dem BSB Adapter in InfluxDB schreiben
# Wenn Ihr die Datenbank nicht vorher angelegt habt, wird dieses automatisch beim Start von Telegraf gemacht
# Sollte InfluxDB nicht auf dem gleichen Systemlaufen, bitte die URL anpassen

[[outputs.influxdb]]
urls = ["http://127.0.0.1:8086"]
namepass = ["bsb"]                   # Wird benötigt, damit nur die Daten vom BSB Adapter in die Datenbank geschrieben werden
database = "Name der Datenbank"      # Diese Daten werden später benötigt, damit die Datenbank in Grafana eingebunden werden kann
username = "USER"
password = "PASSWORT"

## MQTT Daten vom BSB-LAN Adapter bei Mosquitto abfragen ##

[[inputs.mqtt_consumer]]
servers = ["tcp://IP-Adresse-Mosquitto-Server:1883"]

name_override = "bsb"                # Damit werden die Daten markiert und können im output erkannt werden  

# Solltet Ihr in den BSB-LAN Einstellungen den Topic Präfix geändert haben, hier bitte anpassen
topics = ["BSB-LAN/json"]            

# Solltet Ihr in den BSB-LAN Einstellungen den Geräte Präfix geändert haben, hier bitte anpassen
# Über tag_keys könnt Ihr festlegen, welche Werte Ihr säpter in Grafana in den Abfragen verwendet könnt
tag_keys = ["BSB-LAN_name","BSB-LAN_id"]
json_string_fields = ["BSB-LAN_name","BSB-LAN_value","BSB-LAN_desc","BSB-LAN_unit"]
data_format = "json"
```

Wenn Ihr nun Telegraf neustartet, werden die Daten eingesammelt und in die Datenbank geschrieben. Prüfen könnt Ihr dieses, in dem Ihr die CLI von Influx startet und die BSB Datenbank auswählt, die Ihr in der telegraf.conf angegeben habt. Über name_override wird das Measurement (bsb) in der Datenbank festgelegt.

```
root@pi4-x64:~# influx
Connected to http://localhost:8086 version 1.8.10
InfluxDB shell version: 1.8.10
> use bsb
Using database bsb
> select * from bsb order by time desc limit 10
name: bsb
time                BSB-LAN_desc BSB-LAN_error BSB-LAN_id BSB-LAN_name                                BSB-LAN_unit BSB-LAN_value host    topic
----                ------------ ------------- ---------- ------------                                ------------ ------------- ----    -----
1668594761983243473              0             8830       Trinkwassertemperatur-Istwert Oben (B3)     °C           47.7          pi4-x64 BSB-LAN/json
1668594761749700355 Aus          0             8820       Zustand Trinkwasserpumpe                                 0             pi4-x64 BSB-LAN/json
1668594761524161842              0             8774       Vorlauftemperatur-Sollwert resultierend HK2 °C           24.8          pi4-x64 BSB-LAN/json
1668594761288745275              0             8773       Vorlauftemperatur Istwert Heizkreis 2       °C           25.1          pi4-x64 BSB-LAN/json
1668594760870899624 Ein          0             8760       Zustand Heizkreispumpe 2                                 255           pi4-x64 BSB-LAN/json
1668594760657254890              0             8744       Vorlauftemperatur-Sollwert resultierend HK1 °C           18.3          pi4-x64 BSB-LAN/json
1668594760426376270 Aus          0             8730       Zustand Heizkreispumpe 1                                 0             pi4-x64 BSB-LAN/json
1668594760199824672              0             8704       Aussentemperatur gemischt                   °C           12.3          pi4-x64 BSB-LAN/json
1668594759970903815              0             8703       Aussentemperatur gedämpft                   °C           12.2          pi4-x64 BSB-LAN/json
1668594759750286265              0             8700       Aussentemperatur                            °C           12.9          pi4-x64 BSB-LAN/json
```

## Weitere Information zu BSB-LAN in Verbindung mit MQTT

Ich nutze auf meinem Raspberry Pi sehr gerne Docker Container, da dieses m. E. die Administration einfacher macht. Solltet Ihr mit Docker noch keine Erfahrung haben, bitte mal im Internet suchen, da gibt es viel zu entdecken...

Ihr müsst aber nicht Docker verwenden. Ihr könnt die Programme auch originär auf Eurem System installieren. Hierzu bitte im Internet schauen, wie diese zu erfolgen hat.

Um Docker auf dem Raspberry PI zu installieren, könnt Ihr wie folgt vorgehen: 

```
sudo curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker pi
logout
```

### Mosquitto als MQTT Broker

Ich nutze Mosquitto als MQTT Brocker auf meinem Pi. Das Docker Image gibt es hier: https://hub.docker.com/_/eclipse-mosquitto
Dort wird auch beschrieben, wie Ihr das Image nutzen könnt. Wenn Ihr die drei Verzeichnisse und die mosquitto.conf angelegt habt, erweitert Ihr die mosquitto.conf um folgende zwei Zeilen

```
listener 1883
allow_anonymous true
```
Der listener ist wichtig, anderen Falls werden die MQTT Nachrichten im Container nicht sichtbar. Solltet Ihr Mosquitto mit User/Passwort verwenden wollen, bitte im Internet suchen.

Wenn Ihr den Mosquitto Container gestartet hat, könnt Ihr mit folgendem Befehl in die Console des Conatiners wechseln

```
docker exec -it mosquitto /bin/sh
```

Um zu sehen, ob MQTT Nachrichten vom BSB-LAN Adapter kommen, könnt Ihr folgenden Befehl verwenden:

```
mosquitto_sub -t "BSB-LAN/#" -v
```
Abhängig von den gewählten Parametern sollte die Ausgabe dann wie folgt sein:

```
BSB-LAN/status online
BSB-LAN/json {"BSB-LAN":{"id":8304,"name":"Zustand Kesselpumpe (Q1)","value": "255","desc": "Ein","unit": "","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8310,"name":"Kesseltemperatur-Istwert","value": "28.8","desc": "","unit": "°C","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8311,"name":"Kesseltemperatur-Sollwert","value": "34.5","desc": "","unit": "°C","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8314,"name":"Rücklauftemperatur-Istwert","value": "26.8","desc": "","unit": "°C","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8323,"name":"Gebläsedrehzahl","value": "1437","desc": "","unit": "U/min","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8324,"name":"Brennergebläsesollwert","value": "1455","desc": "","unit": "U/min","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8325,"name":"Aktuelle Gebläseansteuerung","value": "12.6","desc": "","unit": "%","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8326,"name":"Brennermodulation","value": "3","desc": "","unit": "%","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8327,"name":"Wasserdruck","value": "2.1","desc": "","unit": "bar","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8330,"name":"Brennerbetriebsstunden Stufe 1","value": "3045","desc": "","unit": "h","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8331,"name":"Brennerstarts Stufe 1","value": "8154","desc": "","unit": "","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8338,"name":"Betriebsstunden Heizbetrieb","value": "2832","desc": "","unit": "h","error": 0}}
BSB-LAN/json {"BSB-LAN":{"id":8339,"name":"BetriebsstundenTrinkwasserbetrieb","value": "2","desc": "","unit": "h","error": 0}}
```

Neben den zyklischen Abfragen könnt Ihr über ein publish auch Parameter abfragen, die nicht in der Parameterliste in den Einstellungen enthalten sind.

Ich nutze hier zwei Terminalfester, eins für den publish (Senden) und eins für die subscripten (Empfangen)...

```
mosquitto_pub -h 192.168.178.34 -m "700" -t BSB-LAN -d 
Client null sending CONNECT
Client null received CONNACK (0)
Client null sending PUBLISH (d0, q0, r0, m1, 'BSB-LAN', ... (3 bytes))
Client null sending DISCONNECT
```

```
mosquitto_sub -t "BSB-LAN/#" -v
BSB-LAN 700
BSB-LAN/MQTT ACK_700
BSB-LAN/json {"BSB-LAN":{"id":700,"name":"Betriebsart","value": "3","desc": "Komfort","unit": "","error": 0}}
```

Der Vollständigkeitshalber noch die Information, ihr habt Über Mosquitto auch die Möglichkeit, die Einstellungen der Heizung zu verändern. Dieses erfolgt ebenfalls  über ein Publish. Hier muss dem Parameter noch ein "S" vorgestellt werden. 

```
mosquitto_pub -h 192.168.178.34 -m "S700" -t BSB-LAN -d 
```

**Anmerkung: Ich übernehme keine Verantwortung, wenn Ihr hier die Einstellungen über Mosquitto verändert. Ich zeige Euch an dieser Stelle nur auf, wie der Befehl lautet**

## Erstellen eines TIG Containers um BSB-LAN MQTT Nachrichten einzusammeln

Neben dem TIG Container benötigt Ihr auch Mosquitto!

Wenn Ihr noch keine TIG Installation auf Eurem System habt, Docker nutzt, könnt Ihr mit hilfe dieses Repo einen eigenen Container erstellen. Im weiteren Verlauf erkläre ich Euch wie.

Aktuell wird der Container mit folgenden Software Versionen erstellt:

| Beschreibung | Wert    |
|--------------|---------|
| InfluxDB     | 1.8.10  |
| Telegraf     | 1.24.3  |
| Grafana      | 9.2.4   |

Diese können jederzeit durch Euch im Dockerfile angepasst werden. **Ein Wechsel auf Influx v2 ist hier nicht möglich**

```
# Default versions
ENV INFLUXDB_VERSION=1.8.10
#ENV CHRONOGRAF_VERSION=1.8.6
ENV GRAFANA_VERSION=9.2.4
ENV TELEGRAF_VERSION 1.24.3-1
```
Um den Container erstellen zu können, müsst Ihr dieses Repo auf Euer System clonen. Die folgenden Befehle beziehen sich auf ein Linux System.

Ich clone die Repos immer ins Verzeichnis /opt/. Ihr könnt dieses auf Eure Ansprüche anpassen...

```
cd /opt/
sudo git clone https://github.com/ronaldn1969/BSB-LAN-MQTT-mit-TIG    
cd BSB-LAN-MQTT-mit-TIG
```

Wenn git nicht verfügbar ist, könnt Ihr es über `sudo apt update && sudo apt install git` installieren.

Jetzt **müsst** Ihr im Verzeichnis telegraf die telegraf.conf um folgende Zeilen erweitern. Bitte beachtet, dass Ihr die IP Adresse Eures MQTT Brokers eintragt.

```
# Daten aus dem BSB Adapter in InfluxDB schreiben
# Wenn Ihr die Datenbank nicht vorher angelegt habt, wird dieses automatisch beim Start von Telegraf gemacht
# Sollte InfluxDB nicht auf dem gleichen Systemlaufen, bitte die URL anpassen

[[outputs.influxdb]]
urls = ["http://127.0.0.1:8086"]
namepass = ["bsb"]                   # Wird benötigt, damit nur die Daten vom BSB Adapter in die Datenbank geschrieben werden
database = "Name der Datenbank"      # Diese Daten werden später benötigt, damit die Datenbank in Grafana eingebunden werden kann
username = "USER"
password = "PASSWORT"

## MQTT Daten vom BSB-LAN Adapter bei Mosquitto abfragen ##

[[inputs.mqtt_consumer]]
servers = ["tcp://IP-Adresse-Mosquitto-Server:1883"]   # Hier Eure IP Adresse eintragen

name_override = "bsb"                # Damit werden die Daten markiert und können im output erkannt werden  

# Solltet Ihr in den BSB-LAN Einstellungen den Topic Präfix geändert haben, hier bitte anpassen
topics = ["BSB-LAN/json"]            

# Solltet Ihr in den BSB-LAN Einstellungen den Geräte Präfix geändert haben, hier bitte anpassen
# Über tag_keys könnt Ihr festlegen, welche Werte Ihr säpter in Grafana in den Abfragen verwendet könnt
tag_keys = ["BSB-LAN_name","BSB-LAN_id"]
json_string_fields = ["BSB-LAN_name","BSB-LAN_value","BSB-LAN_desc","BSB-LAN_unit"]
data_format = "json"
```

Ihr könnt noch weitere Anpassungen vornehmen. z. B. Im Dockerfile die SW Versionen aktualisieren

Oder im Verzeichnis grafana die grafana.ini. Ich habe für mich folgende Anpassungen vorgenommen: 

```
# Diese Anpassungen sind in der INI nicht enthalten!

# Da ich mich nicht jedesmal anmelden möchte, habe ich den Login deaktiviert 

[auth]
# Set to true to disable (hide) the login form, useful if you use OAuth, defaults to false
disable_login_form = true    (line 182)

[auth.anonymous]
# enable anonymous access
enabled = true     (line 186)

# specify role for unauthenticated users
org_role = Admin     (line 192)

# Den Zugriff auf die Grafana URL habe ich auf https umgestellt. Hierzu benötigt Ihr dann eigene Zertifikate
# Diese müsst Ihr in das Verzeichnis kopieren, dass Ihr später beim Start des Containers mit Grafana verbindet

[server]
# Protocol (http or https)
protocol = https     (line 30)

# The http port  to use
http_port = 3003     (line 36)

# The full public facing url you use in browser, used for redirects and emails
# If you use reverse proxy and sub path specify full url (with sub path)
root_url = https://localhost:3003/     (line 47)

# https certs & key file
cert_file = /var/lib/grafana/MyClientCert-pub.crt     (line 59)
cert_key = /var/lib/grafana/MyClientCert-priv.key
```

Wenn Ihr alle Anpassungen vorgenommen habt, könnt Ihr mit `sudo docker build -t integra924 .` den Container erstellen. Den Namen des Containers (intera924) könnt Ihr ebenfalls nach Euren wünschen ändern. Müsst dann aber im weiteren Verlauf in meinen Befehlen Euren Namen nutzen.

## Starten der Container

Ich lege meine Daten alle im Verzeichnis /usr/share/monitoring ab. Dort habe ich dann die benötigten Verzeichnisse angelegt. Solltet Ihr eine andere Verzeichnisstruktur nutzen, müsst Ihr dieses bei den Dockerbefehlen anpassen.

```
/telefraf
/mosquitto/config
/mosquitto/data
/mosquitto/log
/grafana
/Influxdb
```

### Mosquitto Container starten

```
docker run -d \
  --restart=unless-stopped \
  --name mosquitto \
  -p 1883:1883 -p 9001:9001 \
  -v /usr/share/monitoring/mosquitto/config/mosquitto.conf:/mosquitto/config/mosquitto.conf \
  -v /usr/share/monitoring/mosquitto/data:/mosquitto/data \
  -v /usr/share/monitoring/mosquitto/log:/mosquitto/log \
  eclipse-mosquitto
```

Jetzt könnt Ihr, wie weiter oben beschrieben prüfen, ob die MQTT Nachrichten vom BSB-LAN Adapter von Mosquitto empfangen werden können.

### TIG Container starten






alias stopmosquitto='docker stop mosquitto && docker rm mosquitto'
alias mosquittolog='docker logs mosquitto --details'
alias mosquittobash='docker exec -it mosquitto /bin/sh'


docker run  --network host --restart=unless-stopped -d --name integra -v /usr/share/fritzbox/influxdb:/var/lib/influxdb -v /usr/share/fritzbox/grafana:/var/lib/grafana -v /usr/share/fritzbox/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro -v /:/hostfs:ro -v /proc:/hostfs/proc:ro -v /sys:/hostfs/sys:ro -e HOST_ETC=/hostfs/etc -e HOST_PROC=/hostfs/proc -e HOST_SYS=/hostfs/sys -e HOST_VAR=/hostfs/var -e HOST_RUN=/hostfs/run -e HOST_MOUNT_PREFIX=/hostfs integra920:latest'
alias stopinflux='docker stop integra && docker rm integra'
alias influxlog='docker logs integra --details'
alias influxbash='docker exec -it integra /bin/bash'


## Quick Start

To start the container with persistence you can use the following:

```sh
docker run -d \
  --name docker-influxdb-grafana \
  -p 3003:3003 \
  -p 3004:8083 \
  -p 8086:8086 \
  -v /path/for/influxdb:/var/lib/influxdb \
  -v /path/for/grafana:/var/lib/grafana \
  philhawthorne/docker-influxdb-grafana:latest
```

To stop the container launch:

```sh
docker stop docker-influxdb-grafana
```

To start the container again launch:

```sh
docker start docker-influxdb-grafana
```

## Mapped Ports

```
Host		Container		Service

3003		3003			grafana
3004		8083			chronograf
8086		8086			influxdb
```
## SSH

```sh
docker exec -it <CONTAINER_ID> bash
```

## Grafana

Open <http://localhost:3003>

```
Username: root
Password: root
```

### Add data source on Grafana

1. Using the wizard click on `Add data source`
2. Choose a `name` for the source and flag it as `Default`
3. Choose `InfluxDB` as `type`
4. Choose `direct` as `access`
5. Fill remaining fields as follows and click on `Add` without altering other fields

Basic auth and credentials must be left unflagged. Proxy is not required.

Now you are ready to add your first dashboard and launch some queries on a database.

## InfluxDB

### Web Interface (Chronograf)

Open <http://localhost:3004>

```
Username: root
Password: root
Port: 8086
```

### InfluxDB Shell (CLI)

1. Establish a ssh connection with the container
2. Launch `influx` to open InfluxDB Shell (CLI)

[buymeacoffee-icon]: https://www.buymeacoffee.com/assets/img/guidelines/download-assets-sm-2.svg
[buymeacoffee]: https://www.buymeacoffee.com/philhawthorne

[grafana-version]: https://img.shields.io/badge/Grafana-7.2.0-brightgreen
[influx-version]: https://img.shields.io/badge/Influx-1.8.2-brightgreen
[chronograf-version]: https://img.shields.io/badge/Chronograf-1.8.6-brightgreen
