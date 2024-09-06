# ClinicalSite

ClinicalSite von Healex GmbH ist ein Informations- und Verwaltungssystem für organisatorische Daten klinischer Studien. Es unterstützt Sponsoren und Prüfstellen bei der Durchführung qualitativ hochwertiger klinischer Forschung, unter Erfüllung gesetzlicher Organisationsanforderungen.

ClinicalSite dient der digitalen Vernetzung aller an einer Studie beteiligten Parteien, indem die für eine Studie relevanten administrativen Daten zentral im System abgelegt werden und dort von mehreren berechtigten Personen eingesehen und bearbeitet werden können („Cooperative Peer Reviewing“).

- [ClinicalSite](#clinicalsite)
- [Systemanforderungen](#systemanforderungen)
  - [Umgebungseinrichtung](#umgebungseinrichtung)
  - [Docker-Installation](#docker-installation)
  - [Lizenzierung und Download des Docker-Images](#lizenzierung-und-download-des-docker-images)
  - [Hardware-Anforderungen](#hardware-anforderungen)
  - [Client-seitige Anforderungen](#client-seitige-anforderungen)
- [Erste Schritte](#erste-schritte)
- [Konfiguration](#konfiguration)
  - [Umgebungsvariablen](#umgebungsvariablen)
  - [Zwei-Faktor-Authentifizierung](#2fa)
- [SSL über Proxy-Server](#ssl-über-proxy-server)
- [Standard-Login](#standard-login)
- [Container-Shell](#container-shell)
- [Anzeige von Container-Logs](#anzeige-von-container-logs)
- [Übergabe an Key-User](#übergabe-an-key-user)

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
  uploads:

services:
  clinicalsite:
    image: healexsystems/clinicalsite:latest
    container_name: clinicalsite
    init: true
    depends_on:
      - db
    ports:
      - 8080:5000
    volumes:
      - ./config.ini:/app/config.ini:ro
      - dbsocket:/var/run/postgresql     
      - uploads:/uploads

  db:
    image: postgres:15-alpine
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

## Umgebungsvariablen
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
| dir                        | uploads      | Pfad zum Upload Ordner innerhalb des Containers | /tmp | /uploads 
| perm                       | uploads      | Ordner-Zugriffsberechtigung | | 640
| CLASS                      | View::Email transport| Gibt an, welches Email::Sender::Transport-Modul geladen werden soll | DevNull | SMTP
| host                       | View::Email transport| Adresse des E-Mail Servers | | smtp.office365.com
| ssl                        | View::Email transport| Verbindungstyp / Verschlüsselung  | | starttls
| port                       | View::Email transport| Verbindungs-Port; Standart ist 25 für non-SSL, 465 für 'ssl', 587 für 'starttls' | 25 | 587
| timeout                    | View::Email transport| Maximale Teit in Sekunden auf eine Serverrückmeldung  | 120 | 3
| sasl_username              | View::Email transport| Benutzername des E-Mail Kontos zur authentifizierung | | support@example.com
| sasl_password              | View::Email transport| Passwort welches für die Authentifizierung benötigt wird; wird benötigt, falls <sasl_username> gesetzt ist |  | password

Beispiel Template für `config.ini`:

```shell
[run]
disable_template_reload = 1
mock_proxy_host =

[uploads]
dir = /uploads
perm = 640

[app]
instance_badge = #49b #fcfffc Docker
title    = Healex ClinicalSite
support  = support@example.com

[run]
use_ssl = 0
have_reverse_proxy = 0

[View::Email]
sender = support@example.com

[View::Email transport]
CLASS   = SMTP
host    = smtp.example.com
ssl     = starttls
port    = 587
timeout = 3
sasl_username = support@example.com
sasl_password = password
```

## 2FA

Als Methode für eine Zwei-Faktor-Authentifizierung kann optional ein SMS Versand über [seven.io](https://www.seven.io) eingerichtet werden.
Hierfür wird in den Entwicklereinstellungen von seven.io eine neue Anwendung und ein neuer API-Schlüssel erstellt. Dieser ermöglicht einen Zugriff auf die HTTP-API von seven.io.

Das Secret des API-Schlüssels wird in folgende ENV-Variable in der `config.ini` hinterlegt:

```shell
[Model::SMS]
api_key = f1Qery7ShwajQzS5y7uD
```

Desweiteren sind folgende Environment Variablen für den SMS Versand von Bedeutung. Diese sind standardmäßig bereits in der `defaults.ini` enthalten und müssen nicht zwingend in der `config.ini` aufgeführt werden:
```shell
[Model::SMS]
dry_run = 0
sender_name = ClnicalSite
```

# SSL über Proxy-Server

Eine SSL-Verschlüsselung mittels eines Proxy-Servers ist empfohlen und im produktiven Betrieb zwingend. Beispielsweise kann als Proxy-Server [Traefik](https://doc.traefik.io/) verwendet werden.

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

# Übergabe an Key-User

Voraussetzung: E-Mailversand wurde eingerichtet
1. (Eltern-)Organisationseinheit erstellen (ohne Org.-Einheit können keine Zugänge vergeben werden)
3. Key-User als Affilierte in der Org.-Einheit erstellen
4. Einladungs-E-Mails an die Key-User verschicken (Zugänge anlegen)
5. Warten bis die Key-User ihre Zugänge aktiviert haben
6. Key-User unter '/tenant/{tenantID}/edit' als Mandanten-Administratoren hinzufügen