---
title: "Praxis-Beispiel: HA mit GlusterFS, corosync und PacemakerNFS für Apache"
subtitle: "Linux Server Advanced Administration"
author: "![](../images/linux_trainings.png){width=20 height=20} [&copy; Linux-Trainings.de](https://linux-trainings.de/)"
theme: "Luebeck"
colortheme: "whale"
aspectratio: 169
colorlinks: true
urlcolor: gray
linkcolor: gray
---


# Szenario

Wir bauen ein hochverfügbares Webserver-Cluster:

- Daten live spiegeln via GlusterFS über drei Nodes 
- und Pacemaker sorgt für das Failover der IP und des Apache-Dienstes.

# Der technische Rahmen

Als technische Basis für dieses Cluster-Szenario wird KVM-Virtualisierung genutzt mit:

- minimalen Debian-Images
- im privaten Netz 10.0.0.0/24
- mit DNS 10.0.0.1 (KVM-Host zur Namensauflösung)
- sowie Route 10.0.0.1 (KVM-Host als Standard-Gateway)

# Netzwerk & Namensauflösung

Alle Nodes müssen sich blind verstehen und werden daher in die `/etc/hosts`  eingetragen:
```bash
# Voraussetzung: Ausführung auf allen 3 Nodes als root
cat <<EOF >> /etc/hosts
10.0.0.11 node1
10.0.0.12 node2
10.0.0.13 node3
EOF
```

# Speicher-Replikation mit GlusterFS

GlusterFS sorgt dafür, dass die Website-Daten synchron auf allen Nodes liegen.

Installation & Start:
```bash
# Voraussetzung: Paketquellen sind aktuell (apt update)
apt-get install -y glusterfs-server
systemctl enable --now glusterd
```

Cluster-Kopplung:
```bash
# Ausführung: Nur auf node1 ausführen
# Voraussetzung: Glusterd läuft aktiv auf allen Nodes
gluster peer probe node2 && gluster peer probe node3
```

Volume erstellen & starten:
```bash
# Voraussetzung: Brick-Verzeichnis existiert lokal auf allen Nodes
DIR='/gluster/brick1/www'
mkdir -p "$DIR"

# Voraussetzung: Peer-Kopplung im vorherigen Schritt war erfolgreich
gluster volume create web_vol replica 3 node1:"$DIR" node2:"$DIR" node3:"$DIR" force
gluster volume start web_vol
```

# Corosync & Pacemaker

Jetzt bauen wir das HA-Framework auf, das die Ressourcen überwacht und schwenkt.

Installation & Basis-Konfiguration:
```bash
# Ausführung: Ausführung auf allen 3 Nodes
apt-get install -y pacemaker pcs fence-agents
systemctl enable --now pcsd
echo "hacluster:TopSecretPass123" | chpasswd
```

Cluster initialisieren:
```bash
# Ausführung: Nur auf `node1` ausführen
# Voraussetzung: Der pcsd-Dienst läuft auf allen Nodes mit identischem Passwort
pcs cluster auth node1 node2 node3 -u hacluster -p TopSecretPass123
pcs cluster setup --name ha_cluster node1 node2 node3 # erstellt die /etc/corosync/corosync.conf
pcs cluster start --all # hier wird corosync auf allen Nodes gestartet
pcs cluster enable --all
```

Cluster-Grundregeln definieren:
```bash
# Ausführung: Nur auf node1 ausführen
# Voraussetzung: Cluster ist online (Prüfbar via 'pcs status')
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=freeze
```
> Hinweis: `stonith-enabled=false` ist nur für Testumgebungen ohne echte Fencing-Hardware (wie IP-MI) gedacht!

# Apache & Cluster-Ressourcen

Der Webserver darf nicht automatisch vom System gestartet werden → das übernimmt Pacemaker:
```bash
# Ausführung: auf allen 3 Nodes
apt-get install -y apache2
systemctl disable --now apache2
```

Jetzt verknüpfen wir die virtuelle IP (VIP), den GlusterFS-Mount und den Apache-Dienst zu einer einzigen Einheit:
```bash
# Ausführung: Nur auf node1 ausführen

# 1. Virtuelle IP-Ressource anlegen
pcs resource create virtual_ip ocf:heartbeat:IPaddr2 ip=10.0.0.100 cidr_netmask=24 op monitor interval=30s

# 2. GlusterFS-Mount-Ressource anlegen
pcs resource create web_store ocf:heartbeat:Filesystem device="node1:web_vol" directory="/var/www/html" fstype="glusterfs" op monitor interval=20s

# 3. Apache-Ressource anlegen
pcs resource create web_server ocf:heartbeat:apache configfile="/etc/apache2/apache2.conf" op monitor interval=1min

# 4. Ressourcen aneinander binden (Colocation) und Start-Reihenfolge erzwingen (Ordering)
pcs constraint colocation add virtual_ip web_store web_server INFINITY
pcs constraint order virtual_ip then web_store then web_server
```

# Das Testing

Cluster-Status checken:
```bash
pcs status
```

Testseite erstellen:
```bash
# Voraussetzung: Ressourcen sind online und auf dem aktiven Node gemountet
echo "<h1>HA Cluster live von $(hostname)</h1>" > /var/www/html/index.html
curl http://10.0.0.100
```

Failover erzwingen:
```bash
# Simuliere den Ausfall des aktuell aktiven Nodes, indem du dort das Cluster stoppst
# Ausführung auf dem Node, der laut 'pcs status' gerade die VIP hält
pcs cluster stop
```

Seite sollte ohne Unterbrechung erreichbar sein:
```bash
# zeigt dir nun den Hostnamen eines anderen Nodes:
curl http://10.0.0.100
```

# Anregungen für deine Vertiefung

Wenn du das Setup verinnerlicht hast, versuche dich an diesen Upgrades:

- **Echtes Fencing**: Aktiviere STONITH und binde ein virtuelles Fencing-Agent-Modul für KVM (`fence_xvm`) ein.
- **Datenbank-HA**: Versuche, statt des Apache-Webservers eine PostgreSQL-Instanz inklusive synchronem Streaming-Replica-Setup über Pacemaker zu steuern.
- **Split-Brain-Szenario**: Blockiere die Corosync-Ports via `iptables` auf einem Node und beobachte, wie das verbleibende Quorum reagiert.
