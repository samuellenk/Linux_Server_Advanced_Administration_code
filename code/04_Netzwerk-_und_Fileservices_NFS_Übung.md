---
title: "Praxis-Beispiel: NFS Server-Konfiguration"
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

Dieser Abschnitt zeigt ein komplettes, aber kompaktes Beispiel zur Einrichtung eines NFS‑Servers und -Clients.

# Voraussetzung

- zwei Systeme (VMs)
- NFS-Server mit Debian 13
- NFS-Client mit LinuxMint 22
- die Schritte sollten aber (nahezu) deckungsgleich auf anderen Distributionen anwendbar

# Installation

NFS-Server:
```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

NFS-Client:
```bash
sudo apt update
sudo apt install -y nfs-common
```

# Hostnamen setzen und Eintrag in Zonendateien von Bind

- diese Schritte sind nicht zwingend nötig
- aber damit wird eine Namensauflösung vom NFS-Server möglich

Hostnamen **auf NFS-Server** setzen:
```bash
echo "172.16.0.5 nfs.example.net nfs" | sudo tee -a /etc/hosts
echo "nfs.example.net" | sudo tee -a /etc/hostname
```

Eintrag in Zonen-Dateien **auf DNS-Server**:
```bash
cat /etc/bind/zones/db.example.net 
# gekürzte Ausgabe:
# ...
# nfs     IN      A       172.16.0.5
# ...
cat /etc/bind/zones/db.172.16.0 
# gekürzte Ausgabe:
# ...
# 5       IN      PTR     nfs.example.net.
# ...
```
- nicht Anpassung von `serial` und den `rndc reload` vergessen

# Export‑Verzeichnis anlegen

Verzeichnis erstellen:
```bash
sudo mkdir -p /srv/nfs/share
```

Berechtigungen setzen:
```bash
sudo chown nobody:nogroup /srv/nfs/share
sudo chmod 2775 /srv/nfs/share
```

Demo-Inhalt anlegen:
```bash
echo "hallo an NFS" | sudo tee -a /srv/nfs/share/my.txt
```

# `/etc/exports` konfigurieren

Export‑Eintrag hinzufügen:
```bash
sudo bash -c 'cat >> /etc/exports <<EOF
# Voller Zugriff für einen spezifischen Admin-Host
/srv/nfs/share 172.16.0.1(rw,sync,no_subtree_check,no_root_squash)
# Lese-/Schreibzugriff für das gesamte Subnetz (Standard-Sicherheit)
/srv/nfs/share 172.16.0.0/16(rw,sync,no_subtree_check,root_squash)
EOF'
```

# Exporte aktivieren

Exporte neu laden:
```bash
sudo exportfs -a
```

# NFS‑Service starten bzw. neu laden

Service abfragen:
```bash
sudo systemctl status nfs-kernel-server
```

Service starten, falls noch nicht läuft:
```bash
sudo systemctl enable --now nfs-kernel-server
```

Service neu starten, falls er bereits läuft:
```bash
sudo systemctl restart nfs-kernel-server
```

welche Version stellt welche Funktionen bereit:
```bash
sudo rpcinfo -p localhost
```
- `mountd`ist nicht für NFS-Version 4 vorhanden, weil es direkt im NFS integriert ist

# Überprüfung auf dem Server

Export‑Liste anzeigen:
```bash
# kurze Ausgabe:
sudo exportfs
# ausführliche Ausgabe:
sudo exportfs -v
```

verfügbare Exporte des eigenen Hosts:
```bash
sudo showmount -e localhost
```

# Überprüfung auf dem Client

## auf einem Linux-Client

Mount‑Verzeichnis anlegen:
```bash
sudo mkdir -p /mnt/nfs/share
```

manuell mounten:
```bash
sudo mount -t nfs nfs.example.net:/srv/nfs/share /mnt/nfs/share
```
- schlägt der `mount` fehl, dann die [Installation](#Installation) für den Client prüfen

Mount prüfen:
```bash
mount | grep nfs
```

Inhalt ansehen:
```bash
ls -l /mnt/nfs/share
```

## auf einem Windows-Client

- über Windows Features den NFS Client aktivieren
- dann den Share aufrufen:
```cmd
mount.exe -o anon 192.168.111.102:/srv/nfs/share N:
```

# Automatischer Mount über `/etc/fstab`

zunächst wieder aushängen:
```bash
sudo umount /mnt/nfs/share
```

Zeile hinzufügen:
```bash
sudo bash -c 'echo "nfs.example.net:/srv/nfs/share  /mnt/nfs/share  nfs  defaults,_netdev  0  0" >> /etc/fstab'
```

Eintrag prüfen:
```bash
sudo mount -av
```
- sollte anzeigen `/mnt/nfs/share : successfully mounted`

# NFS-Troubleshooting

1. Dienste & RPC-Check: Prüfe, ob `nfs-server` und `rpcbind` laufen. Teste die Erreichbarkeit vom Client aus mit `rpcinfo -p <Server-IP>`.
2. Export-Validierung: Nutze `exportfs -v` auf dem Server, um zu sehen, welche Pfade mit welchen Optionen tatsächlich aktiv sind.
3. Berechtigungen (UID/GID): Stelle sicher, dass die User-IDs auf Client und Server identisch sind und prüfe die `root_squash` Einstellungen in der `/etc/exports`.
4. Netzwerk & Ports: Verifiziere, dass Port 2049 (und bei NFSv3 zusätzlich Port 111) nicht durch die Firewall blockiert wird.
5. Log-Analyse: Checke `journalctl -xeu nfs-server` oder `/var/log/syslog` direkt nach einem fehlgeschlagenen Mount-Versuch auf präzise Fehlermeldungen.

# Weitere Tests

Schreiben testen:
```bash
echo "test" | sudo tee /mnt/nfs/share/test.txt
```

Lesen testen:
```bash
cat /mnt/nfs/share/test.txt
```

NFS‑Statistiken zeigen:
```bash
sudo nfsstat -s
```

# Mount-Unit und Automount erstellen mit Dependency

- zunächst **Eintrag in `/etc/fstab` löschen**
- und wieder aushängen: `sudo umount /mnt/nfs/share`
- überprüfen mit `mount | grep nfs`, wo (spätestens) jetzt nicht mehr auftauchen sollte

Datei `/etc/systemd/system/mnt-nfs-share.mount` anlegen:
```bash
[Unit]
Description=NFS Share Mount

[Mount]
What=nfs.example.net:/srv/nfs/share
Where=/mnt/nfs/share
Type=nfs
Options=defaults,_netdev

[Install]
WantedBy=multi-user.target
```

Datei `/etc/systemd/system/mnt-nfs-share.automount` anlegen:
```bash
[Unit]
Description=Automount für NFS Share (aktiviert sich bei Zugriff)
# Stellt sicher, dass das Netzwerk bereit ist
After=network-online.target
Wants=network-online.target

[Automount]
Where=/mnt/nfs/share
# Timeout nach Untätigkeit (optional): Trennt die Verbindung nach 5 Min Inaktivität
# TimeoutIdleSec=300

[Install]
WantedBy=multi-user.target
```

Systemd neu laden:
```bash
sudo systemctl daemon-reload
```

AutoMount aktivieren:
```bash
sudo systemctl enable --now mnt-nfs-share.automount
```

Test des Auto-Mount:
```bash
ls /mnt/nfs/share
```

# Aufräumen

Exporte entfernen:
```bash
echo "" | sudo tee /etc/exports
```

Export (re-export/all) permanent entfernen:
```bash
sudo exportfs -ra
```

Überprüfung:
```bash
sudo exportfs -v
```
- sollte keinen Export mehr anzeigen

# Weiterführende Befehle/Themen

- ACLs in NFSv4 konfigurieren: `nfs4_setfacl` / `getfacl`
- ...
