---
title: "Praxis-Beispiel: Active Directory Services mit Samba"
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

Diese Übung enthält ein vollständiges Beispiel zur Nutzung vom Samba als Active Directory Domain Controller (AD DC).

# Voraussetzung

- ein System (VM) mit Debian 13 als Server - hier wird SLES/RedHat nicht unterstützt
- für den Server wir din diesem Beispiel die statische IP `192.168.122.2` und der Hostname `addc-01.example.net` verwendet
- weiteres System mit Windows 11, das in die Domäne aufgenommen werden kann

# Pakete installieren

```bash
apt update
apt install samba libpam-heimdal heimdal-clients ldb-tools winbind libpam-winbind smbclient libnss-winbind bind9 dnsutils
```
- die Abfrage vom Kerberos-Realm während der Installation einfach mit [Enter] bestätigen
- was die Datei `/etc/krb5.conf` anlegen wird

# Dienste stoppen

Bevor die Konfiguration startet, müssen die Standard-Dienste angehalten werden:
```bash
systemctl stop smbd nmbd winbind
systemctl disable smbd nmbd winbind
```

# verwendete Daten

Das sind die Daten, die in unserem Setup zum Einsatz kommen:

| Key                | Beschreibung                  | Value            |
|--------------------|-------------------------------|------------------|
| Realm              | meist DNS-Domain-Name         | `EXAMPLE.NET`    |
| NetBIOS-Domainname | Adresse für NetBIOS-Protokoll | `EXAMPLE`        |
| Funktion           | nicht Standalone/Member       | Domaincontroller |
| DNS-Server         | DNS-Server im Samba AD DC     | `BIND9_DLZ`      |

# Anpassung von `/etc/hosts`

Das als DC verwendete System muss lokal auflösbar sein:
```bash
echo "192.168.122.2 addc-01.example.net addc-01" | sudo tee -a /etc/hosts
```

# Verwendung von samba-tool

Überprüfung, ob samba-tool installiert ist:
```bash
samba-tool -V
```

Hilfe erhalten:
```bash
samba-tool -h
```

```bash
samba-tool -V
```

# Provisioning der Domain

Alte Konfiguration entfernen, da sie beim Setup als DC sonst stört:
```bash
rm /etc/samba/smb.conf
```

Domäne einrichten:
```bash
samba-tool domain provision --interactive
```
- hier musst du einige Daten angeben, die wir unter [verwendete Daten](#verwendete-daten) aufgeführt haben
- auch die Ausgabe ist recht umfangreich

das sollte am Ende der Ausgabe vom Provisioning auftauchen:
```bash
Once the above files are installed, your Samba AD server will be ready to use
Server Role:    active directory domain controller
Hostname:       addc-01
NetBIOS Domain: EXAMPLE
DNS Domain:     example.net
DOMAIN SID:     S-1-5-21-2492729547-2181408212-3299912776
```

danach die generierte krb.conf an die richtige Stelle kopieren:
```bash
cp /var/lib/samba/privat/krb5.conf /etc/
```

# DNS-Konfiguration

Forwarder angeben in der `/etc/bind/named.conf.options`:
```bash
forwarders {
  8.8.8.8;
};
tkey-gssapi-keytab "/var/lib/samba/bind-dns/dns.keytab";
```
- `forwarders` ist der Upstream-DNS-Server
- `tkey-gssapi-keytab` ist für die Kerberos-Authentifizierung von Bind9

in der Datei `/etc/bind/named.conf.local` die vorherige laden:
```text
include "/var/lib/samba/bind-dns/named.conf";
```
- Zonendateien werden in diesem Fall direkt im AD verwaltet und damit müssen sie hier nicht erstellt werden

# Anpassung von AppArmor

[ToDo - S. 543]

# Aktivierung des Domaincontrollers (DC)

Start vom Samba-AD-DC Dienst:
```bash
systemctl unmask samba-ad-dc
systemctl enable samba-ad-dc
systemctl start samba-ad-dc
```

# Benutzer- und Gruppenverwaltung

Benutzer anlegen:
```bash
samba-tool user create <benutzername>
```

Gruppe anlegen:
```bash
samba-tool group add <gruppenname>
```

Mitglied hinzufügen:
```bash
samba-tool group addmembers <gruppenname> <benutzername>
```

Passwort-Richtlinien (PSO) anzeigen:
```bash
samba-tool domain passwordsettings show
```
