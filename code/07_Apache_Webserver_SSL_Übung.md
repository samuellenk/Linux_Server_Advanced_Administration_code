---
title: "Praxis-Beispiel: Apache-Konfiguration für HTTPS"
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

Diese Übung erweitert das Beispiel zur Nutzung vom Apache-Server um SSL.

# Voraussetzung

- ein System (VM) mit **einer festen IP-Adresse**
- spezielle verwendet wurde hier Apache-Server auf Debian 13
- die Schritte sollten aber (nahezu) deckungsgleich auf anderen Distributionen anwendbar sein

# Installation

Debian-Familie:
```bash
sudo apt update
sudo apt install apache2 -y
sudo apt install curl -y
sudo apt install openssl -y
```
- das Paket `openssl` wird meist bereits installiert sein

# Status und Version prüfen

Dienst:
```bash
sudo systemctl status apache2
sudo apache2 -v
```

OpenSSL:
```bash
openssl -v
```

# Verzeichnis für Zertifikate anlegen

```bash
sudo mkdir /etc/apache2/ssl
cd !$
```

# Zertifikat erstellen

Web-Ordner:
```bash
sudo openssl req -new -x509 -keyout myserver.pem -out myserver.pem -days 365 -nodes
```
- Fragen können beliebig beantwortet werden
- aber **Common Name muss dem Webangebot/FQDN entsprechen**
- in diesem Beispiel `www.example.net`
- wird später mit `https://www.example.net` aufgerufen

# Ports prüfen

```bash
cat /etc/apache2/ports.conf
```

sollte bereits das enthalten:
```bash
<IfModule ssl_module>
    Listen 443
</IfModule>

<IfModule mod_gnutls.c>
    Listen 443
</IfModule>
```
- wenn nicht, dann einfach (zusätzlich) einfügen

# Modul aktivieren

bereits vorhandenes Modul aktivieren:
```bash
sudo a2enmod ssl
```

Server neu starten:
```bash
sudo systemctl restart apache2
```

Server beherrscht jetzt HTTPS:
```bash
sudo lsof -i | grep https
```

# Ordner und Webseiten erstellen

Web-Ordner:
```bash
sudo mkdir -p /var/www/example.net
```

Web-Inhalte:
```bash
echo "<h1>HTTPS Site</h1>" | sudo tee /var/www/example.net/index.html
```

# Virtuellen Hosts mit HTTPS erstellen

Konfiguration mit HTTPS in `/etc/apache2/sites-available/ssl_example.net.conf`:
```bash
<VirtualHost *:443>
    ServerName www.example.net
    ServerAdmin webmaster@example.net
    DocumentRoot /var/www/example.net/
    ErrorLog /var/log/apache2/error.log
    CustomLog /var/log/apache2/example.net.log combined
    SSLEngine On
    SSLCertificateFile /etc/apache2/ssl/myserver.pem
    <Directory /var/www/example.net/>
        Options Indexes FollowSymLinks
    </Directory>
</VirtualHost>
```

# Konfiguration testen

```bash
sudo apachectl configtest
# erwartete Ausgabe:
Syntax OK
```

# Seiten aktivieren

```bash
sudo a2ensite ssl_example.net.conf
sudo systemctl restart apache2
```

# Seiten testen

Aufruf auf dem **Server**:
```bash
curl -k https://www.example.net
```
- mit `-k` (`--insecure`) akzeptiert `curl` auch das selbst-signierte Zertifikat

Aufruf auf dem **Client**:
```bash
firefox https://www.example.net
```
- im Browser gibt es eine Warnung, die man aber ignorieren kann
- und es lässt sich das Zertifikat anzeigen
