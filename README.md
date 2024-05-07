# ClinicalSite

Das ClinicalSite der Healex GmbH ist ein Informations- und Verwaltungssystem für organisatorische Daten klinischer Studien mit dem Ziel, Sponsor und Prüfstellen bei der Durchführung qualitativ hochwertiger klinischer Forschung an Menschen und bei der Erfüllung gesetzlicher Organisationsanforderungen zu unterstützen. 

Das Healex ClincialSite dient der digitalen Vernetzung aller an einer Studie beteiligten Parteien und ist darauf angelegt, dass die für eine Studie relevanten administrativen Daten zentral im System abgelegt sind und dort von mehreren berechtigten Personen eingesehen und bearbeitet werden können („Cooperative Peer Reviewing“). 

- [Systemanforderungen](#systemanforderungen)
    * [Umgebungseinrichtung](#Umgebungseinrichtung)
    * [Docker installation](#docker-installation)
    * [Lizensierung & Image Download](#lizensierung-und-download-des-docker-images)
    * [Clientseitige Anforderungen](#clientseitige-anforderungen)
- [Erste Schritte](#Erste-Schritte)
- [Umgebungsvariablen](#Umgebungsvariablen)
- [Docker Secrets](#Docker-Secrets)
- [Datenbank Konfiguration](#Initiale-Datenbank-konfiguration)
- [SSL Proxy Server](#ssl-proxy-server)
- [Login](#Standart-Login)
- [Container Shell](#Zugriff-auf-die-Container-Shell)
- [Logs](#anzeige-von-container-logs)

# Systemanforderungen
## Umgebungseinrichtung

ClinicalSite ist containerisiert und lässt sich mit einer OCI-kompatiblen Runtime betreiben. Der Betrieb bspw. in Docker ist problemlos möglich und wird aufgrund bisheriger Erfahrungen von Healex empfohlen.  

## Docker installation

Hierfür wird eine Docker Umgebung benötigt (https://www.docker.com/products/container-runtime).
Mit dieser können die Hosts (unabhängig davon ob real, virtual, lokal oder remote) ausgeführt und angesteuert werden.

Installieren Sie hierfür die Docker Engine: https://docs.docker.com/get-docker/

## Lizensierung und Download des Docker Images

Es wird ein Zugang zum privaten Docker Healex Repository benötigt.
Wenden Sie sich hierfür an  <support@healex.systems> um die Lizenzbedingungen zu besprechen. 
Sobald die Lizenz eingerichtet ist, erhalten Sie ihren Kontonamen, mit welchem der Zugriff zum Docker Repository ermöglicht wird. 

## Hardware-Anforderungen

| Service                    | vCPU    | RAM    | Festplattenspeicher |
|----------------------------|---------|--------|---------------------|
| ClinicalSite               | 2       | 4 GB   | 10 GB               |

Diese Mindestwerte eignen sich für Tests, Prototypen, Pre-PROD und erfahrungsgemäß für den anfänglichen Betrieb der Produktionsumgebung. 
Abhängig von der Entwicklung der realen Nutzung können sich hiervon abweichende Systemanforderungen ergeben. 
Die Hardwareanforderungen sollten vom hauseigenen Betrieb überwacht und angepasst werden. 

## Clientseitige Anforderungen

  - Keine speziellen Hardwareanforderungen notwendig
  - Aktivierung von JavaScript und Cookies (i.d.R. Browser Standarteinstellungen)
  - Standartkonformer Browser. Aktuelle (nicht älter als 1 Jahr) Versionen des
      * Google Chrome
      * Mozilla Firefox
      * Microsoft Edge

# Erste Schritte

Am einfachsten lässt sich ClinicalSite über `docker compose` starten. Hierfür werden folgende Services benötigt:

  - clinicalsite: Applikations Container
  - db: Postgres Datenbank Container

Beispiel für eine minimale Konfiugration:

```yaml 
version: "3.7"

volumes:
  db:

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

  db:
    image: postgres:14-bookworm
    container_name: cs-db
    environment:
      POSTGRES_DB: cs
      POSTGRES_USER: cs
      POSTGRES_PASSWORD: example
    volumes:
      - db:/var/lib/postgresql/data

```

# Konfiguration

`defaults.ini` und `config.ini`
-------------------------------

Hier erfolgt die Konfiguration der Anwendung

Die im Image vorliegende Datei `defaults.ini` enthält Werkseinstellungen und wird immer geladen.
Sie stellt Teil der Anwendung dar und sollte keine lokalen Änderungen haben.

Anschliessend wird `config.ini` geladen, falls vorhanden.
Die Angaben in dieser Datei können somit Werkseinstellungen übersteuern.
Das ist in Produktion zwingend nötig, weil die Werkseinstellungen
auf Entwickler-Instanzen zugeschnitten sind.


| Umgebungsvariable          | Abschnitt    | Erforderlich | Kommentar/Beschreibung                                                     | Standart Wert |  Beispiel                               |
|----------------------------|--------------|--------------|----------------------------------------------------------------------------|---------------|-----------------------------------------|
| disable_template_reload    | run          |              | Wenn eingeschaltet, werden die Templates nur beim Start der Anwendung eingelesen, nicht mehr im laufenden Betrieb. Sollte in Produktion eingeschaltet sein.| 0              | 0                                       |
| have_reverse_proxy         | run          |              |Gibt an, ob ein Reverse-Proxy eingesetzt wird (in Produktion immer; in Entwickler-Instanzen nie; für Test-Instanzen kann es variieren) | 0 | 0 |
| readonly                   | run          |              |  Wenn eingeschaltet, überspringt die Anwendung beim Start Änderungen an den Daten in der Datenbank (Momentan bezieht sich das nur aufs Cleanup des Gatekeeper-Moduls). |               |                                        |
| mock_proxy_host            | run          |              | Wenn gesetzt, läuft die Anwendung im Proxy-Modus und kann nur unter dem angegebenen Hostnamen angesprochen werden |  clinicalsite.org             |                                        |
| use_ssl                    | run          |              | ??? |  1             | 1  |
| title                      | app          |              | Titel der Anwendung, u.a. in jedem Seitentitel.                            |               | Beispiel GmbH                                       |
| instance_badge             | app          |              | Vordergrund- und Hintergrundfarbe sowie Beschriftung der Instanz-Plakette. Sie wird oben rechts im Layout der Anwendung dargestellt, um verschiedene Installationen optisch voneinander unterscheidbar zu machen. | black yellow Dev              |                                        |
| support                    | app          |              | Haupt-Support-EMail-Adresse des Systems                    |               | support@example.com |
| superusers                 | app          |              | ???                    | 1              | 1 |
| superclient                | app          |              | ???                    | 1              | 1 |
| supertenant                | app          |              | ???                    | 1              | 1 |
| secret                     | signatures   |              | Das HMAC-Secret zum Signieren von Cookies und Links (z.B. in Einrichtungsmails)                    |               |                                        |
| timeout                    | signatures   |              | Dauer der Gültigkeit von signierten Cookies und Links                    |               |                                        |
| parallelism                | authenticator|              | ???                    | 4               | 4                                       |
| dir                        | uploads      |              | Verzeichnis zur Speicherung von durch Benutzer hochgeladene Dateien        | /tmp | /data/upload |
| perm                       | uploads      |              | Unix-Berechtigungsbits (als Oktal-Zahl), mit den hochgeladene Dateien abgelegt werden      | 640 | 640                                          |
| api_key                    | Model::SMS   |              | Der API-Key für die SMS-API      |               |                                        |
| dry_run                    | Model::SMS   |              | Entspricht dem `debug`-Parameter der SMS-Versand-API: der API-Call wird durchgeführt, simuliert den SMS-Versand aber nur.      |               |                                        |
| sender_name                | Model::SMS   |              | Absendernummer bzw. -name. Max. 16 Ziffern bzw. 11 Zeichen     | 0  | 0 |
| sender                     | View::Email  |              | Enveloper-Sender, der in vom System versandten EMails angegeben wird     |               |                                        |
| CLASS                      | View::Email transport  |              | Email::Sender::Transport::-Modul, das zum EMail-Versand verwendet wird. In Produktion muss `config.ini` einen sinnvollen Wert einstellen. Die restliche Konfiguration (ausser dem `CLASS`-Schlüssel) wird als zweiter Parameter an die `easy_transport`-Methode von `Email::Sender::Util` übergeben.     | DevNull | DevNull |

# Initiale Datenbank konfiguration

Beim initialisieren der Datenbank werden standartmäßig folgende [lokalen-Einstellungen](https://www.postgresql.org/docs/current/locale.html#LOCALE) verwendet und müssen von der Datenbank unterstützt werden:

```shell
LC_COLLATE "de_DE.UTF-8"
LC_CTYPE   "de_DE.UTF-8"
```
Für das Docker Image kann dies beispielsweise über ein Dockerfile konfiguriert werden:

```shell
RUN localedef -i de_DE -c -f UTF-8 -A /usr/share/locale/locale.alias de_DE.UTF-8
ENV LANG de_DE.utf8
```

# SSL Proxy-Server
Eine SSL-Verschlüsselung mittels eines Proxy Servers ist empfohlen und im produktiven Betrieb zwingend.

# Standart Login

die initialen Zugangsdaten lauten:

Benutzer: muster <br>
Passwort: admin5

# Zugriff auf die Container-Shell

Mit dem Befehl `docker exec` können Befehle innerhalb des laufenden Containers ausgeführt werden. 
Mit der folgenden Befehlszeile wird eine Bash Shell im Container geöffnet.

```shell
docker exec -it docker Container-ID /bin/bash
```

# Anzeige von Container Logs

```shell
docker logs Container-ID
```