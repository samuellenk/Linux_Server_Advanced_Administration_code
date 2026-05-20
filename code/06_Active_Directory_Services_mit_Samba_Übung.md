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

Diese Übung enthält ein vollständiges Beispiel zur Nutzung vom **Samba als Active Directory Domain Controller (AD DC)**.

# Voraussetzung

- ein System (VM) mit Debian 13 als Server - hier wird SLES/RedHat nicht unterstützt
- für den Server wird in diesem Beispiel die statische IP `192.168.122.2` und der Hostname `addc-01.example.net` verwendet
- ein weiteres System mit Windows 11, das in die Domäne aufgenommen werden kann

# verwendete Daten

Diese Daten kommen in unserem Setup zum Einsatz:

| Key                | Beschreibung                  | Value            |
|--------------------|-------------------------------|------------------|
| Realm              | meist DNS-Domain-Name         | `EXAMPLE.NET`    |
| NetBIOS-Domainname | Adresse für NetBIOS-Protokoll | `EXAMPLE`        |
| Funktion           | nicht Standalone/Member       | Domaincontroller |
| DNS-Server         | DNS-Server im Samba AD DC     | `BIND9_DLZ`      |

# Pakete installieren

installiert Samba, Kerberos (Heimdal), DNS (`bind9`):
```bash
apt update
apt install samba libpam-heimdal heimdal-clients ldb-tools winbind libpam-winbind smbclient libnss-winbind bind9 dnsutils
```
- die Abfrage vom Kerberos-Realm während der Installation einfach mit [Enter] bestätigen
- was die Datei `/etc/krb5.conf` anlegen wird, die wir später überschreiben

# Anpassung von `/etc/hosts`

Das als DC verwendete System muss lokal auflösbar sein:
```bash
echo "192.168.122.2 addc-01.example.net addc-01" | sudo tee -a /etc/hosts
```

# Verwendung von `samba-tool`

Überprüfung, ob `samba-tool` installiert ist:
```bash
samba-tool -V
```

Hilfe erhalten:
```bash
samba-tool # zeigt automatisch die Hilfe an
samba-tool -h # zeigt explizit die Hilfe an
samba-tool provision -h # zeigt Hilfe für einen Sub-Befehl an
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

Das sollte am Ende der Ausgabe vom Provisioning auftauchen:
```
Once the above files are installed, your Samba AD server will be ready to use
Server Role:    active directory domain controller
Hostname:       addc-01
NetBIOS Domain: EXAMPLE
DNS Domain:     example.net
DOMAIN SID:     S-1-5-21-2492729547-2181408212-3299912776
```

danach die generierte `krb.conf` an die richtige Stelle kopieren:
```bash
cp /var/lib/samba/private/krb5.conf /etc/
```

# DNS-Konfiguration

Forwarder angeben im Block `options` in der `/etc/bind/named.conf.options`:
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

## Anpassung von AppArmor

Das AppArmor-Profil muss für Bind den Zugriff auf die Datenbanken des AD erlauben.

Profil `/etc/apparmor.d/usr.sbin.named` sollte folgenden Inhalt haben:
```bash
/etc/bind/** r,
/var/lib/bind/** rw,
/var/lib/bind/ rw,
/var/cache/bind/** lrw,
/var/lib/samba/** rwmk,
/usr/lib/x86_64-linux-gnu/** rwmk,
/dev/urandom rwmk,
```
- wird heute meist gegeben sein

# Neustart von Bind9

Nach Anpassung im AppArmor-Profile den Neustart von Bind nicht vergessen:
```bash
sudo systemctl restart bind9
```

im Log sollten danach folgende Zeilen sichtbar sein:
```bash
...: samba_dlz: Loading 'AD DNS Zone' using driver dlopen
...: samba_dlz: started for DN DC=example,DC=net
...: samba_dlz: starting configure
...: samba_dlz: configured writeable zone 'example.net'
...: samba_dlz: configured writeable zone '_msdcs.example.net'
```

diese Meldungen im Log finden:
```bash
# im traditionellen Syslog:
grep 'samba_dlz' /var/log/syslog
# im neueren Journal:
journalctl --grep 'samba_dlz'
```

# Stoppen der normalen Samba-Dienste

Im Default sind die Dienste für den Domaincontroller noch nicht aktiv und die anderen laufen.

unnötige Dienste stoppen und deaktivieren:
```bash
sudo systemctl disable --now nmbd smbd winbind
```

# Aktivierung des Domaincontrollers (DC)

Start vom Samba-AD-DC Dienst:
```bash
systemctl unmask samba-ad-dc
systemctl enable samba-ad-dc
systemctl start samba-ad-dc
```

Überprüfung nach dem Start des DC:
```bash
ps ax | grep 'samba'
```

# Nutzung vom eingebauten DNS

Jetzt noch dem AD-DC-Server anpassen, sodass der eingebaute DNS-Server genutzt wird.

In neueren Versionen von Debian kannst das im passenden `netplan.yaml` hinterlegen:
```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false
      accept-ra: false
      link-local: []
      addresses:
        - 192.168.122.2/24
      routes:
        - to: default
          via: 192.168.122.1
      nameservers:
        search:
          - example.net
        addresses:
          - 192.168.122.2
```
- je nach Setup vom Server kann es sein, dass du hier statt `netplan` die `/etc/network/interfaces` oder `nmcli` nutzen musst

Änderungen übernehmen:
```bash
netplan apply
```

Ergebnis prüfen:
```bash
netplan status
```

# Neustart

Ein Reboot des AD-DC-Servers ist an dieser Stelle angebracht. Damit wird der saubere Start aller Komponenten bestätigt.

# Test des Domaincontroller (DC)

alle Dienst sollten wieder laufen:
```bash
ps ax | grep 'samba'
```

oder Lauscher abfragen:
```bash
ss -tlpn | awk '{print $1" "$2" "$3" "$4}'
```
- gibt alle Dienste aus, die der DC bereitstellt

IPv6 deaktivieren über `interfaces` in `/etc/samba/smb.conf`:
```bash
[global]
  ...
  interfaces = 192.168.122.2
  ...
```

Änderung wirksam werden lassen:
```bash
systemctl restart samba-ad-dc
```
- jetzt sollte der `ss`-Befehl von oben nur noch die Ausgaben für IPv4 beinhalten

# Test des DNS-Servers

```bash
host addc-01
host -t SRV _kerberos._tcp.example.net
host -t SRV _ldap._tcp.example.net
host -t SRV _gc._tcp.example.net
```

Der Reverse-Lookup funktioniert noch nicht, da im Bind nur die Forward-Zone angelegt wird.

Reverse-Lookup-Zone anlegen:
```bash
# Zone erzeugen:
samba-tool dns zonecreate addc-01 122.168.192.in-addr.arpa
# DC-Server eintragen:
samba-tool dns add addc-01 122.168.192.in-addr.arpa 2 PTR addc-01.example.net
```

Reverse-Zone prüfen:
```bash
host 192.168.122.2
```

# Test von Kerberos

Ticket erzeugen:
```bash
kinit administrator
```
- fragt nach dem Passwort für den User `administrator`

Ticket anzeigen:
```bash
klist
```
- zeigt Daten zum vorhandenen Ticket an
- wichtig: einmal erstelltes Ticket kann jeder nutzen, solange es gültig ist = also immer den Rechner sperren

Ticket löschen:
```bash
kdestroy
```

# Test der Verbindung

erste Anmeldung:
```bash
smbclient -L addc-01
# Password for [administrator@EXAMPLE.NET]:
```
- einfach [Enter] drücken beim Passwort

Beispiel-Ausgabe:
```bash
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        sysvol          Disk      
        netlogon        Disk      
        IPC$            IPC       IPC Service (Samba 4.22.8-Debian-4.22.8+dfsg-0+deb13u1)
SMB1 disabled -- no workgroup available
```

Anmeldung mit Nutzung vom Kerberos-Ticket:
```bash
smbclient -L addc-01 -N
```
- fragt nicht nach dem Passwort

# Zeitserver einrichten

Für die Authentifizierung am DC ist es wichtig, dass für Kerberos eine gültige, aktuelle Zeit gesetzt ist.

Paket installieren, falls nötig:
```bash
apt install chrony
```

Konfiguration in `/etc/chrony/chrony.conf`:
```bash
local stratum 10

pool 0.pool.ntp.org iburst prefer
pool 1.pool.ntp.org iburst prefer

driftfile /var/lib/chrony/chrony.drift

logdir /var/log/chrony
log measurements statistics tracking

deny all
allow 127.0.0.1
```

Socket für die Bereitstellung an Clients muss korrekt berechtigt sein:
```bash
sudo ls -ld /var/lib/samba/ntp_signd/
# erwartete Ausgabe:
drwxr-x--- 2 root _chrony 4096 May 19 15:12 /var/lib/samba/ntp_signd/
# Änderung bei Bedarf:
chgrp _chrony /var/lib/samba/ntp_signd/
chrmod 750 /var/lib/samba/ntp_signd/
```

NTP-Dienst neu starten:
```bash
systemctl restart chronyd
```

# Abschluss der Einrichtung vom AD-DC

Damit die Einrichtung von Samba als AD-DC abgeschlossen.

Jetzt können User, Gruppen usw. in dem Verzeichnis zentral gepflegt werden.

# Verwaltung von Gruppen und Usern

Beachte, dass du hier keine lokalen Namen verwenden solltest.

Unter Linux werden lokale User/Gruppen werden immer vor denen aus dem AD verwendet.

# Gruppen managen

Gruppen anzeigen:
```bash
samba-tool group list
```
- Ausgabe ist bereits recht umfangreich
- bitte diese **Gruppen nicht löschen**

Gruppen-Mitglieder anzeigen:
```bash
samba-tool group listmembers administrators
```

Gruppe anlegen:
```bash
samba-tool group add datengruppe
```

Windows-Gruppe abfragen:
```bash
wbinfo -g
wbinfo -g | grep -i 'daten'
```

# Anpassung der `/etc/nsswitch.conf`

Gruppe noch **nicht sichtbar** unter Linux:
```bash
getent group datengruppe
```

Gruppen-Abfrage in `/etc/nsswitch.conf` aktivieren:
```bash
passwd:         files winbind
group:          files winbind
```

Gruppe jetzt **sichtbar** unter Linux:
```bash
getent group datengruppe
```
- nur `getent group` wird nie funktionieren für AD-Gruppen

# User managen

Mitglieder zu Gruppe hinzufügen:
```bash
samba-tool group addmembers datengruppe "Domain Users"
```

User auflisten:
```bash
samba-tool user list
```
- auch hier keine der schon vorhandenen Einträge löschen

Benutzer anlegen:
```bash
samba-tool user create Samuel P@ssw0rd --given-name=Samuel --surname=Lenk
```
- kann fehlschlagen mangels Passwort-Qualität

Passwort interaktiv mitgeben:
```bash
samba-tool user create franky
```
- fragt dann 2x das Passwort ab

Windows-User abfragen:
```bash
wbinfo -u
wbinfo -u | grep -i 'Samuel'
getent passwd Samuel
```

User deaktivieren:
```bash
 samba-tool user disable samuel
```
- oft der Löschung vorzuziehen, weil auch ein identisch angelegter User trotzdem nie wieder die gleiche SID bekommt

User reaktivieren:
```bash
samba-tool user enable samuel
```

Passwort für User ändern:
```bash
samba-tool user setpassword samuel
```

User-Daten anzeigen:
```bash
samba-tool user show samuel
```

User-Daten bearbeiten:
```bash
samba-tool user edit samuel
```

# Passwort-Richtlinien (PSO) managen

Passwort-Richtlinien (PSO) anzeigen:
```bash
samba-tool domain passwordsettings show
```

Passwort-Richtlinien (PSO) ändern:
```bash
samba-tool domain passwordsettings set --min-pwd-age=2
samba-tool domain passwordsettings set --max-pwd-age=90
```

Anlegen eines PSO:
```bash
samba-tool domain passwordsettings pso create verwaltung 1 --min-pwd-age=5
```

Zuweisen eines PSO:
```bash
samba-tool domain passwordsettings pso apply verwaltung datengruppe
```

Anzeige vom Ergebnis:
```bash
samba-tool group show datengruppe
```

# Windows-Client in Domäne aufnehmen

DNS-Server korrekt setzen:
```PowerShell
$NetworkAdapter = Get-NetRoute -DestinationPrefix "0.0.0.0/0" | Select-Object -First 1
Set-DnsClientServerAddress -InterfaceIndex $NetworkAdapter.InterfaceIndex -ServerAddresses "192.168.122.2"
```

Befehl für die PowerShell:
```PowerShell
Add-Computer -DomainName "example.net" -Credential "example.net\Administrator" -Restart
```
- kann eine Weile dauern
- und erfordert danach einen Neustart

# Remote Server Administration Tools (RSAT)

als Domänen-Admin anmelden:
```PowerShell
Start-Process powershell -Credential "example\administrator" -WorkingDirectory "C:\"
```

Features abfragen:
```PowerShell
Get-WindowsCapability -Online | Where-Object {$_.Name -like "Rsat*"}
```

nötige Features aktivieren:
```PowerShell
# RSAT DNS-Servertools
Add-WindowsCapability -Online -Name "Rsat.Dns.Tools~~~~0.0.1.0"
# RSAT DNS-Servermanager
Add-WindowsCapability -Online -Name "Rsat.ServerManager.Tools~~~~0.0.1.0"
# RSAT AD-Domain-Services
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
# RSAT GPO
Add-WindowsCapability -Online -Name "Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0"
```

installierte Features abfragen:
```PowerShell
Get-WindowsCapability -Online | Where-Object {$_.State -eq "Installed"}
```

Ab jetzt kannst du auch auf dem Windows-Client in der Domäne mit den installierten Tools die AD-Instanz managen. Die Programme dazu findest du unter *Start → Alle Programme → Windows Tools*
