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
  - [ClinicalSite](#clinicalsite-1)
  - [Datenbank](#datenbank)
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
volumes:
  db:
  uploads:

services:
  clinicalsite:
    image: healexsystems/clinicalsite:20250604-013ab7
    container_name: clinicalsite
    cap_drop:
      - ALL
    depends_on:
      db:
        condition: service_healthy
    ports:
      - 8080:5000
    volumes:
      - uploads:/uploads
      - type: bind
        source: ./config.ini
        target: /app/config.ini
        read_only: true
      - type: bind
        source: ./config.bash
        target: /app/config.bash
        read_only: true

  db:
    image: postgres:15-alpine
    container_name: cs-db
    environment:
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: psql -U csrun -d clinicalsite
      interval: 1s
      start_period: 10s
      timeout: 60s
      retries: 60
    volumes:
      - db:/var/lib/postgresql/data
      - type: bind
        source: ./init.sql
        target: /docker-entrypoint-initdb.d/init.sql
        read_only: true
```

# Konfiguration

## ClinicalSite
Der Service clinicalsite wird hauptsächlich über die Datei `config.ini` konfiguriert.

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

Beispiel Template für eine `config.ini`:

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

## Datenbank

Die Datenbank Konfiguration muss zueinander passend in der `config.bash`, `init.sql` und `compose.yml` vorgenommen werden. Die `init.sql` erstellt den ClinicalSite-Anwendungs-Benutzer und die Datenbank, das DB-Schema spielt die ClinicalSite Anwendung ein. 

Startet der Docker Datenbank Service mit einem leeren Volume, wird einmalig die `init.sql` ausgeführt und somit eine Datenbank und ein Datenbankbenutzer mit Passwort angelegt. Der Postgres Docker Service benötigt zwingend die Umgebungsvariable `POSTGRES_PASSWORD` welche in der `compose.yml` gesetzt werden muss. Außerdem müssen die Werte aus dem compose-`healthcheck`-Abschnitt mit dem Benutzer und dem Datenbanknamen aus der `init.sql` übereinstimmen.

In der Datei `config.bash` werden die Umgebungsvariablen der Anwendung ClinicalSite definiert, damit sie sich mit der Datenbank verbinden kann. Dementsprechend müssen Benutzername `$PGUSER` und Passwort `$PGPASSWORD` sowie der Datenbankname `$PGDATABASE` mit den Werten aus der `init.sql` übereinstimmen.


### `init.sql`
```shell
CREATE USER "$PGUSER" PASSWORD '$PGPASSWORD';
DROP DATABASE IF EXISTS "$PGDATABASE";
CREATE DATABASE "$PGDATABASE" WITH
	TEMPLATE   template0
	OWNER      "$PGUSER"
	ENCODING   UTF8
	LOCALE_PROVIDER icu
	ICU_LOCALE "de-DE"
	LC_COLLATE "de_DE.UTF-8"
	LC_CTYPE   "de_DE.UTF-8"
;
```

Beispiel für eine init.sql:
```shell
CREATE USER "csrun" PASSWORD 'csrun';
DROP DATABASE IF EXISTS "clinicalsite";
CREATE DATABASE "clinicalsite" WITH
	TEMPLATE   template0
	OWNER      "csrun"
	ENCODING   UTF8
	LOCALE_PROVIDER icu
	ICU_LOCALE "de-DE"
	LC_COLLATE "de_DE.UTF-8"
	LC_CTYPE   "de_DE.UTF-8"
;
```

`config.bash`

Umgebungsvariablen welche in der `config.bash` gesetzt werden können, sind [hier](https://www.postgresql.org/docs/15/libpq-envars.html) zu entnehmen:

Die folgende Tabelle zeigt die gängisten Umgebungsvariablen:

| Umgebungsvariable          | Beschreibung | Beispiel
|----------------------------|--------------|----------
| PGHOST                     |Hostadresse der Datenbank | cs-db
| PGUSER                     |Name des Postgres Benutzers. Dieser muss mit dem Benutzer in der init.sql und dem Benutzer welcher im Datenbank-healtcheck in der compose.yml festgelegt wird, übereinstimmen| csrun
| PGPASSWORD                 |Passwort des Postgres Benutzers welcher über `PGUSER` definiert wurde. Dieser Wert muss mit dem Wert in der init.sql übereinstimmen | csrun
| PGDATABASE                 |Datenbankname | clinicalsite

Beispiel Template für eine `config.bash`:
```shell
export PGHOST=cs-db
export PGUSER=csrun
export PGPASSWORD=csrun
export PGDATABASE=clinicalsite
```

`compose.yml`
| Umgebungsvariable          | Beschreibung | Beispiel
|----------------------------|--------------|----------
| POSTGRES_PASSWORD          |Superuser Passwort for PostgreSQL. Diese muss zwingend gesetzt werden.    | ayfwWL4kx9aNz1M

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