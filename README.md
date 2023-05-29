# APEX with Tomcat

APEX-Umgebung mittels ORDS deployt in einem Tomcat-Container zur Verfügung stellen.

![Tomcat and ORDS](img/ORDS%20and%20Tomcat.png)

Für diese Übungen werden Downloads von Oracle benötigt, die einen Oracle-Zugang benötigen.

* [Oracle APEX 23.1](https://www.oracle.com/tools/downloads/apex-downloads/)
* [Oracle REST Data Services (ORDS)]()

Alle anderen Tools können über _Homebrew_ auf dem Mac installiert werden.

Für das Oracle XE Image wird ein Account bei Oracle benötigt, sodass der Zugriff auf die Docker-Registry für den Download des Images möglich ist.

## Installation / Deinstallation Tomcat

Installation von Tomcat auf der Kommandozeile mit Homebrew

```bash
brew install tomcat@9
```

Deinstallation von Tomcat 9

```bash
brew uninstall tomcat@9
```

## Oracle Database

Oracle XE 21.3.0 wird als Docker-Image innerhalb von Kubernetes (Rancher) verwendet.

Anschließend kann die Deployment-Konfiguration für die Kubernetes-Umgebung mittels

```bash
kubectl apply -f xe-pod.yaml
```

eingerichtet werden.
Das Log zur Erstellung der Datenbank kann über `kubectl logs -f <PODNAME>` eingesehen werden.

### Datenbanken

|Datenbank|Typ
|---|---
|Global Database (CDB)|XE
|Pluggable Database (PDB)|XEPDB1

### Benutzer und Passwörter

Oracle Name|Benutzer|Passwort|
|---|---|---|
|SYS|SYS|GoodMorning2023|
|APEX ADMIN|ADMIN|!Apex2023
|APEX Public User|APEX_PUBLIC_USER|!Apex2023
|APEX Listener|APEX_LISTENER|!Apex2023
|APEX REST Public User|APEX_REST_PUBLIC_USER|!Apex2023

## SQL Plus installieren

Auf einem Mac kann SQLplus mittels Homebrew installiert werden. Hierfür sind die folgenden Befehle in einem Terminal abzusetzen:

```bash
brew tap InstantClientTap/instantclient
brew install instantclient-basic
brew install instantclient-sqlplus
```

## APEX Installation

APEX entpacken und mittels _sqlplus_ die Installation durchführen.
Hierfür mit der Kommandaozeile in das Verzeichnis stellen, in dem die SQL-Dateien entpackt worden sind.

Mit dem Parameter _/i/_ bei dem Installationsskript wird der Ordner für die
spätere Ablage der statischen Ressourcen angegeben.

```sqlplus
sqlplus sys@localhost:1521/XE as sysdba
alter session set container = xepdb1;
@apexins.sql SYSAUX SYSAUX TEMP /i/
@apxchpwd.sql
```

In der gleichen sqlplus-Session können die Benutzer für APEX geändert werden.
Dies gescheht über das Script `apxchpwd.sql` im oben gezeigten Listing.

Die Benutzer sind in der Tabelle unter _Benutzer und Passwörter_ gelistet.

Freischaltung des APEX Schemas:

```sqlplus
ALTER USER APEX_PUBLIC_USER ACCOUNT UNLOCK;
ALTER USER APEX_PUBLIC_USER IDENTIFIED BY "!Apex2023";
```

## ORDS Installation

ORDS entpacken und mittels _java_ über die Kommandozeile die Installation durchführen. In den folgenden Listings wird `$ORDS_CONFIG` immer stellvertretend für den Ablageort des Konfigurationsverzeichnis verwendet.

Das Konfigurationsverzeichnis sollte immer außerhalb des Produktverzeichnis liegen, sodass mit Erscheinen einer neuen Version eine vereinfachte Migration durchgeführt werden kann.

Es wird keine Umgebungsvariable angelegt, da in einem späteren Schritt ein neues _ords.war_ mit dem Pfad zur Konfiguration erstellt wird.

```bash
./bin/ords --config $ORDS_CONFIG install
```

Der Parameter `config` verweist auf das Konfigurationsverzeichnis für ORDS.

### Installationsparameter

Defaults übernehmen bis zur Eingabe Datenbankservicename eingeben.

Dann mit den folgenden Eingaben fortfahren:

* Datenbankservicename eingeben [orcl]: XEPDB1
* Geben Sie den Administratorbenutzernamen ein: sys

Weiter mit den Defaults bis zur Eingabe der statischen APEX-Ressourcen.
An dieser Stelle den globalen Pfad zu den APEX Ressourcen angeben (bspw. "/Users/bberg/Development/apex/tomcat-apex/apex-23/apex/images")

**HINWEIS:** Dies wird nur vorübergehend benötigt.
Sobald ORDS im Tomcat deployt ist, werden die Ressourcen an einen anderen Ort kopiert.

### Kopieren der statischen APEX-Ressourcen

Im Konfigurationsverzeichnis (_config_) muss noch ein Ordner *doc_root* erstellt werden, der die statischen Ressourcen
bereitstellen kann.
In diesen Ordner werden die Images aus dem APEX Produktordner kopiert und der Ordner _images_ nach _i_ umbenannt.

### Test der Installation

Die Installation kann vorübergehend in der Standalone-Variante von ORDS getestet werden:

```bash
./bin/ords --config $ORDS_CONFIG serve`
```

Für `config` ist der Ordner anzugeben, der für die Ablage der Konfigurationsdateien bei der Installation angegeben worden ist.

Im Browser sollte unter _http://localhost:8080_ eine Seite zu sehen sein.

### Deployment von ORDS in Tomcat

Über die ORDS Kommandozeilentools wird ein neues WAR-Archiv erstellt, dass den Pfad zum Konfigurationsverzeichnis enthält:

```bash
./bin/ords --config $ORDS_CONFIG war ../ords.war
```

Weitere Schritte:

Unter _webapps_ einen neuen Ordner mit dem Namen _i_ erstellen und die statischen APEX-Ressourcen dorthin kopieren.

In den Beispielen wir davon ausgegangen, dass wir bereits im Ordner _webapps_ der Tomcat-Installation stehen.

```bash
cp -R /Users/bberg/Development/apex/tomcat-apex/apex-23/apex/images/ ./i/
```

Das neu erzeugte `ords.war` in den Ordner _webapps_ kopieren.

```bash
cp -R /Users/bberg/Development/apex/tomcat-apex/ords.war .
```

## Starting / Stopping Tomcat

```bash
brew services start tomcat@9
/usr/local/opt/tomcat@9/bin/catalina run
```

```bash
brew services stop tomcat@9
/usr/local/opt/tomcat@9/bin/catalina stop
```

## Troubleshooting

Status der Benutzer in der DB prüfen:

```sql
SELECT 
    username, lock_date, expiry_date, account_status 
FROM 
    dba_users 
WHERE 
    username IN ( 
        'ANONYMOUS', 
        'APEX_PUBLIC_USER', 
        'APEX_LISTNER', 
        'APEX_REST_PUBLIC_USER', 
        'ORDS_METADATA', 
        'ORDS_PUBLIC_USER' 
    ) 
ORDER BY 1;
```