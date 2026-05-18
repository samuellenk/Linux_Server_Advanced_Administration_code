---
title: "Praxis-Beispiel: Grundlagen der Apache-Konfiguration"
subtitle: "Linux Server Advanced Administration"
author: "![](../images/linux_trainings.png){width=20 height=20} [&copy; Linux-Trainings.de](https://linux-trainings.de/)"
theme: "Luebeck"
colortheme: "whale"
aspectratio: 169
colorlinks: true
urlcolor: gray
linkcolor: gray
---

# Einleitung

Diese Übung enthält ein vollständiges Beispiel zur Nutzung vom Apache-Server.

# Voraussetzung

- ein System (VM) mit **zwei IP-Adressen**
- Apache-Server auf Debian 13
- die Schritte sollten aber (nahezu) deckungsgleich auf anderen Distributionen anwendbar sein

# Installation

Debian-Familie:
```bash
sudo apt update
sudo apt install apache2
```

Redhat-Familie:
```bash
sudo dnf install httpd
```
- hier heisst dann auch der Dienst entsprechend `httpd`
- ebenso kann sich der Ort für die Konfiguration und das Web-Verzeichnis unterscheiden
- für einen Überblick die Inhalte vom Paket auflisten: `rpm -ql httpd`

# Status und Version prüfen

Dienst:
```bash
sudo systemctl status apache2
## ggf. starten:
sudo systemctl enable --now apache2
```

Apache-Befehl zur Versions-Abfrage:
```bash
sudo apache2 -v
```

# Basic Configuration Directives

Haupt-Datei:
```bash
sudo nano /etc/apache2/apache2.conf
```

kann enthalten:

- `ServerName` mit dem sich Apache selbst identifiziert
- `DocumentRoot` als Ordner für vom Webserver bereitgestellte Dateien
- `ErrorLog` und `CustomLog` als Log-Dateien

Daten anhängen:
```bash
echo "ServerName web.example.net" | sudo tee -a apache2.conf
```

# Ordner und Webseiten erstellen

Web-Ordner:
```bash
sudo mkdir -p /var/www/site{1..2}/public_html
```

Web-Inhalte:
```bash
echo "<h1>Site 1</h1>" | sudo tee /var/www/site1/public_html/index.html
echo "<h1>Site 2</h1>" | sudo tee /var/www/site2/public_html/index.html
```

# Ports prüfen

```bash
grep 'Listen' /etc/apache2/ports.conf
# sollte ausgeben:
Listen 80
```

# Virtuelle Hosts erstellen

Konfiguration in `/etc/apache2/sites-available/site1.conf`:
```bash
<VirtualHost 172.16.0.4:80>
    ServerName site1
    DocumentRoot /var/www/site1/public_html

    ErrorLog ${APACHE_LOG_DIR}/site1_error.log
    CustomLog ${APACHE_LOG_DIR}/site1_access.log combined

    <Directory /var/www/site1/public_html>
        Require all granted
    </Directory>
</VirtualHost>
```

- die *normale* VirtualHost-Zeile sieht so aus: `<VirtualHost *:80>`
- und bedeutet, dass Apache auf allen IPs lauscht

Konfiguration in `/etc/apache2/sites-available/site2.conf` mit **Basic Auth**:
```bash
<VirtualHost 172.16.0.5:80>
    ServerName site2
    DocumentRoot /var/www/site2/public_html

    ErrorLog ${APACHE_LOG_DIR}/site2_error.log
    CustomLog ${APACHE_LOG_DIR}/site2_access.log combined

    <Directory /var/www/site2/public_html>
        AuthType Basic
        AuthName "Restricted Area"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Directory>
</VirtualHost>
```

# IP-Adressen hinzufügen

nur **vorübergehend** hinzufügen:
```bash
sudo ip addr add 172.16.0.4/16 dev enp1s0
sudo ip addr add 172.16.0.5/16 dev enp1s0
```

oder **dauerhaft** hinzufügen in `/etc/network/interfaces`:
```bash
allow-hotplug enp1s0
iface enp1s0 inet static
        address 172.16.0.4/16
        gateway 172.16.0.1

iface enp1s0 inet static
        address 172.16.0.5/16
```

# IP-Adressen anzeigen

```bash
ip -br a s enp1s0
ip     a s enp1s0
```

# Netzwerk-Manager neu starten

```bash
sudo systemctl restart networking
```

# Einträge in /etc/hosts vornehmen

Auszug:
```bash
127.0.0.1       localhost site1 site2
```

die Seiten am DNS-Server bekannt machen:

```bash
[ToDo]
```

# User für Authentifizierung erstellen

```bash
# neu erstellen:
sudo htpasswd -c /etc/apache2/.htpasswd user1
# hinzufügen:
sudo htpasswd /etc/apache2/.htpasswd user2
# löschen:
sudo htpasswd -D /etc/apache2/.htpasswd user1
# anzeigen:
cat /etc/apache2/.htpasswd
```

# Konfiguration testen

```bash
sudo apachectl configtest
# erwartete Ausgabe:
Syntax OK
```

# Seiten aktivieren

```bash
sudo a2ensite site1.conf
sudo a2ensite site2.conf
sudo systemctl reload apache2
```

# Seiten testen

Aufruf auf dem **Server**:
```bash
curl 172.16.0.4
curl 172.16.0.5 # zeigt Fehler (401)
```

Aufruf auf dem **Client**:
```bash
# Site 1:
curl 172.16.0.4
firefox 172.16.0.4
# Site 2:
curl 172.16.0.5 # zeigt Fehler (401)
firefox 172.16.0.5 # fragt nach User & Passwort
```

# Logging & Troubleshooting

Apache lauscht:
```bash
netstat -tlnp | grep apache
ss -tlnp | grep :80
```

Logs finden und ansehen:
```bash
sudo ls -Al /var/log/apache2/
sudo tail -f /var/log/apache2/site1_access.log
sudo tail /var/log/apache2/site2_error.log
```

Logging konfigurieren:
```bash
LogFormat "%h %l %u %t \"%r\" %>s %b" lpic2
CustomLog ${APACHE_LOG_DIR}/site1_access.log lpic2
```

# Module verwalten

Module anzeigen:
```bash
sudo apachectl -M
ls /etc/apache2/mods-enabled/
```

Modul aktivieren:
```bash
sudo a2enmod rewrite
sudo systemctl restart apache2
```

Modul deaktivieren:
```bash
sudo a2dismod rewrite
sudo systemctl restart apache2
```

# Zugriff mittels `.htaccess`

Gründe:

- `.htaccess` erlaubt verzeichnisbasierte Apache-**Konfiguration ohne Root-Zugriff**, ist aber **langsamer** als zentrale Konfiguration und funktioniert nur mit `AllowOverride`
- allgemein also nur bei Shared Hosting sinnvoll

die `.htaccess` aktivieren:
```bash
<Directory /var/www/site2/public_html>
    AllowOverride AuthConfig # Zeile für .htaccess
    Require all granted
</Directory>
```

Datei selbst erstellen:
```bash
sudo nano /var/www/site2/public_html/.htaccess
```

möglicher Inhalt der Datei:
```bash
AuthType Basic
AuthName "Restricted Area"
AuthUserFile /etc/apache2/.htpasswd
Require valid-user
Options -Indexes # Verzeichnis-Listing deaktivieren
Redirect 301 /oldpage.html /newpage.html # Weiterleitung
Require ip 172.16.0.0/24 # Zugriffsbeschränkung
```
