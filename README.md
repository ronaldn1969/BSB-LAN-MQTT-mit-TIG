Ausgang ist das Repo von Phil Hawthorne, ich habe es für meine Bedürfnisse angepasst. Bedeutet, ich habe Chronograf entfernt und Telegraf hinzugefügt.

Anmerkung von Phil: This is a Docker image based on the awesome [Docker Image with Telegraf (StatsD), InfluxDB and Grafana](https://github.com/samuelebistoletti/docker-statsd-influxdb-grafana) from [Samuele Bistoletti](https://github.com/samuelebistoletti).


## Anpassungen in der telegraf.conf, damit die MQTT Daten vom BSB-LAN in eine InfluxDB geschrieben werden können

Voraussetzung ist, dass Ihr schon Telegraf, InfluxDB, Grafana und Mosquitto auf Eurem System laufen habt. Wenn nicht, könnt Ihr weiter unten entnehmen, wie Ihr Euch einen einen TIG Container erstellen könnt und Mosquitto installiert.

Anmerkung: Ich nutze InfluxDB v1.8.10. Es kann sein, dass die Anpassungen für InfluxDB v2.x nicht passen.

```
# Daten aus dem BSB Adapter in InfluxDB schreiben
# Wenn Ihr die Datenbank nicht vorher angelegt habt, wird dieses automatisch beim Start von Telegraf gemacht
# Sollte InfluxDB nicht auf dem gleichen Systemlaufen, bitte die URL anpassen

[[outputs.influxdb]]
urls = ["http://127.0.0.1:8086"]
namepass = ["bsb"]                   # Wird benötigt, damit nur die Daten vom BSB Adapter in die Datenbank geschrieben werden
database = "Name der Datenbank"
username = "USER"
password = "PASSWORT"

## MQTT Daten vom BSB-LAN Adapter bei Mosquitto abfragen ##

[[inputs.mqtt_consumer]]
servers = ["tcp://IP-Adresse-Mosquitto-Server:1883"]

 # Damit werden die Daten markiert und können im output erkannt werden
name_override = "bsb"               

# Solltet Ihr in den BSB-LAN Einstellungen Topic Präfix geändert haben, hier bitte anpassen
topics = ["BSB-LAN/json"]            

# Solltet Ihr in den BSB-LAN Einstellungen Geräte Präfix geändert haben, hier bitte anpassen
# Über tag_keys könnt Ihr festlegen, welche Werte Ihr säpter in Grafana in den Abfragen verwendet könnt
tag_keys = ["BSB-LAN_name","BSB-LAN_id"]
json_string_fields = ["BSB-LAN_name","BSB-LAN_value","BSB-LAN_desc","BSB-LAN_unit"]
data_format = "json"
```







The main point of difference with this image is:

* Persistence is supported via mounting volumes to a Docker container
* Grafana will store its data in SQLite files instead of a MySQL table on the container, so MySQL is not installed
* Telegraf (StatsD) is not included in this container

The main purpose of this image is to be used to show data from a [Home Assistant](https://home-assistant.io) installation. For more information on how to do that, please see my website about how I use this container.

| Description  | Value   |
|--------------|---------|
| InfluxDB     | 1.8.2   |
| ChronoGraf   | 1.8.6   |
| Grafana      | 7.2.0   |

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
