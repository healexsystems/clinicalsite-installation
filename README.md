# ClinicalSite

ClinicalSite von Healex GmbH ist ein Informations- und Verwaltungssystem für organisatorische Daten klinischer Studien. Es unterstützt Sponsoren und Prüfstellen bei der Durchführung qualitativ hochwertiger klinischer Forschung, unter Erfüllung gesetzlicher Organisationsanforderungen.

ClinicalSite dient der digitalen Vernetzung aller an einer Studie beteiligten Parteien, indem die für eine Studie relevanten administrativen Daten zentral im System abgelegt werden und dort von mehreren berechtigten Personen eingesehen und bearbeitet werden können („Cooperative Peer Reviewing“).

- [Systemanforderungen](#systemanforderungen)
    * [Umgebungseinrichtung](#umgebungseinrichtung)
    * [Docker-Installation](#docker-installation)
    * [Lizenzierung und Download des Docker-Images](#lizenzierung-und-download-des-docker-images)
    * [Client-seitige Anforderungen](#client-seitige-anforderungen)
- [Erste Schritte](#erste-schritte)
- [Datenbank-Konfiguration](#datenbank-konfiguration)
- [SSL über Proxy Server](#ssl-über-proxy-server)
- [Standard-Login](#standard-login)
- [Container-Shell](#container-shell)
- [Anzeige von Container-Logs](#anzeige-von-container-logs)

# Systemanforderungen

## Umgebungseinrichtung

ClinicalSite ist containerisiert und lässt sich mit einer OCI-kompatiblen Runtime betreiben, bspw. mit Docker, welches dazu aufgrund bisheriger Erfahrungen von Healex empfohlen wird.

## Docker-Installation

Hierfür wird eine Docker Umgebung benötigt (https://www.docker.com/products/container-runtime). 
Mit dieser können die Hosts (unabhängig davon, ob real, virtuell, lokal oder remote) ausgeführt und angesteuert werden.

Installieren Sie hierfür die [Docker-Engine](https://docs.docker.com/get-docker/).

## Lizenzierung und Download des Docker-Images

Es wird ein Zugang zum privaten Healex Docker Repository benötigt.  
Wenden Sie sich hierfür an <support@healex.systems> um die Lizenzbedingungen zu besprechen. Sobald die Lizenz eingerichtet ist, erhalten Sie ihren Kontonamen, mit welchem der Zugriff zum Docker Repository ermöglicht wird.

## Hardware-Anforderungen

| Service                    | vCPU    | RAM    | Festplattenspeicher
|----------------------------|---------|--------|---------------------
| ClinicalSite               | 2       | 4 GB   | 10 GB

Diese Mindestwerte eignen sich für Tests, Prototypen, Pre-PROD und erfahrungsgemäß für den anfänglichen Betrieb der Produktionsumgebung. 
Abhängig von der Entwicklung der realen Nutzung können sich hiervon abweichende Systemanforderungen ergeben. 
Die Hardwareanforderungen sollten vom hauseigenen Betrieb überwacht und angepasst werden.

## Client-seitige Anforderungen

  - Keine speziellen Hardwareanforderungen notwendig
  - Aktivierung von JavaScript und Cookies (i.d.R. Browser Standardeinstellungen)
  - Standardkonformer Browser: aktuelle Versionen (nicht älter als 1 Jahr) des
      * Google Chrome
      * Mozilla Firefox
      * Microsoft Edge
      * Safari

# Erste Schritte

Am einfachsten lässt sich ClinicalSite über `docker compose` starten. Hierfür werden folgende Services benötigt:

  - clinicalsite: Applikationscontainer
  - db: Postgres Datenbank-Container

Beispiel für eine minimale Konfiguration:

```yaml
version: "3.7"

volumes:
  db:
  dbsocket:

services:
  clinicalsite:
    image: healexsystems/clinicalsite:latest
    container_name: clinicalsite
    depends_on:
      - db
    ports:
      - 8080:5000
    volumes:
      - ./config.ini:/app/config.ini:ro
      - dbsocket:/var/run/postgresql     

  db:
    image: postgres:14-bookworm
    container_name: cs-db
    environment:
      POSTGRES_DB: cs
      POSTGRES_USER: cs
      POSTGRES_PASSWORD: example
    volumes:
      - db:/var/lib/postgresql/data
      - dbsocket:/var/run/postgresql      
      - ./init.sql:/docker-entrypoint-initdb.d/01-schema.sql:ro
```

# Konfiguration

Die Anwendung wird über die Datei `config.ini` konfiguriert.

| Umgebungsvariable          | Abschnitt    | Beschreibung | Standard-Wert | Beispiel
|----------------------------|--------------|--------------|---------------|----------
| have_reverse_proxy         | run          | Gibt an, ob die Anwendung hinter einem Reverse-Proxy betrieben wird (sollte in Produktion immer der Fall sein) | 0 | 0
| use_ssl                    | run          | Wenn eingeschaltet, erzwingt die Anwendung die Verwendung von HTTPS für URLs und Cookies | 1 | 1
| readonly                   | run          | Wenn eingeschaltet, überspringt die Anwendung beim Start Änderungen an den Daten in der Datenbank.
| title                      | app          | Titel der Anwendung, u.a. in jedem Seitentitel. | Healex ClinicalSite | ClinicalTest
| instance_badge             | app          | Vordergrund- und Hintergrundfarbe sowie Beschriftung der Instanz-Plakette. Sie wird oben rechts im Layout der Anwendung dargestellt, um verschiedene Installationen optisch voneinander unterscheidbar zu machen. | #49b #fcfffc Docker
| support                    | app          | Haupt-Support-E-Mail-Adresse des Systems | support@clinicalsite.org | support@example.com
| superusers                 | app          | Durch Leerzeichen getrennte Personen-IDs der Benutzer mit Systemverwaltungszugriff | 1 | 1
| superclient                | app          | Client-ID eines externen Systems, das Zugriff auf alle Reports bekommt | 1 | 1
| supertenant                | app          | Mandanten-ID des Mandanten, dessen Hilfetexte für Formularfelder in allen anderen Mandanten angezeigt werden | 1 | 1
| secret                     | signatures   | HMAC-Secret zum Signieren von Cookies und Links (z.B. in Einrichtungsmails) | | 88cd2108b5347d9
| timeout                    | signatures   | Dauer der Gültigkeit von signierten Cookies und Links in Stunden | 72
| api_key                    | Model::SMS   | API-Key für die SMS-API | | f1Qery7ShwajQzS5y7uD
| dry_run                    | Model::SMS   | Entspricht dem `debug`-Parameter der SMS-Versand-API: der API-Call wird durchgeführt, simuliert den SMS-Versand aber nur.
| sender_name                | Model::SMS   | Absendernummer bzw. -name. Max. 16 Ziffern bzw. 11 Zeichen | ClnicalSite
| sender                     | View::Email  | Envelope-Sender, der in vom System versandten EMails angegeben wird | support@clinicalsite.org | support@example.com


Template für `config.ini`:

```shell
[run]
disable_template_reload = 1
mock_proxy_host =

[uploads]
dir = /app/data/uploads

[app]
instance_badge = #49b #fcfffc Docker

[run]
use_ssl = 0
have_reverse_proxy = 0
```

# Datenbank-Konfiguration

Beim Initialisieren der Datenbank werden standardmäßig folgende [lokalen Einstellungen](https://www.postgresql.org/docs/current/locale.html#LOCALE) verwendet und müssen von der Datenbank unterstützt werden:

```shell
LC_COLLATE "de_DE.UTF-8"
LC_CTYPE   "de_DE.UTF-8"
```
Für das Docker-Image kann dies beispielsweise über ein Dockerfile konfiguriert werden:

```shell
FROM postgres:14-bookworm
RUN localedef -i de_DE -c -f UTF-8 -A /usr/share/locale/locale.alias de_DE.UTF-8
ENV LANG de_DE.utf8
```

# SSL über Proxy-Server

Eine SSL-Verschlüsselung mittels eines Proxy-Servers ist empfohlen und im produktiven Betrieb zwingend.

# Standard-Login

Die initialen Zugangsdaten lauten:

Benutzer: muster <br>
Passwort: admin5

# Container-Shell

Mit dem Befehl `docker exec` können Befehle innerhalb des laufenden Containers ausgeführt werden.
Mit der folgenden Befehlszeile wird eine Bash-Shell im Container geöffnet.

```shell
docker exec -it docker Container-ID /bin/bash
```

# Anzeige von Container-Logs

```shell
docker logs Container-ID
```
