---
title: "Praxis-Beispiel: OpenLDAP-Server konfigurieren"
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

Diese Übung enthält ein vollständiges Beispiel zur Nutzung vom LDAP-Server.

# Voraussetzung

- ein System (VM) mit Debian 13
- die Schritte sollten aber (nahezu) deckungsgleich auf anderen Distributionen anwendbar sein

# Installation

OpenLDAP-Pakete:
```bash
sudo apt update
sudo apt install slapd ldap-utils -y
```

# Initiale Konfiguration

entweder so:
```bash
sudo dpkg-reconfigure slapd
# dann Prompts folgen: Domain (z.B. example.net) und Admin-Passwort
```

oder mit `ldapmodify`:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f config.ldif
```

Inhalt von `config.ldif`:
```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=example,dc=net

dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=example,dc=net

dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}passwort_hash_hier
```

Passwort-Hash erzeugen:
```bash
sudo slappasswd
```
- fragt das Passwort 2x ab und zeigt dann den Hash an, der in `config.ldif` verwendet wird

# Verzeichnis befüllen

grundlegende Struktur:
```bash
sudo ldapadd -x -D cn=admin,dc=example,dc=net -W -f base.ldif
```

Inhalt von `base.ldif`:
```ldif
dn: dc=example,dc=net
objectClass: top
objectClass: dcObject
objectClass: organization
o: Example Inc.
dc: example

dn: ou=people,dc=example,dc=net
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=example,dc=net
objectClass: organizationalUnit
ou: groups
```

# Daten abfragen

so kannst du die Daten abfragen:
```bash
ldapsearch -x -H ldapi:/// -D "cn=admin,dc=example,dc=net" -W -b "dc=example,dc=net" "(objectClass=*)"
ldapsearch -x -b "dc=example,dc=net"
```
- gibt alle Objekte aus
- im Abschnitt zum [LDAP-Client](./code/exercises/210_ldap_client_03.md) geht es ausführlicher um die Suche

# User und Gruppen administrieren

User anlegen:
```bash
sudo ldapadd -x -D cn=admin,dc=example,dc=net -W -f user.ldif
```

Inhalt von `user.ldif`:
```ldif
dn: uid=john,ou=people,dc=example,dc=net
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: John Doe
sn: Doe
uid: john
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/john
```

Eintrag ändern:
```bash
sudo ldapmodify -x -D cn=admin,dc=example,dc=net -W -f modify.ldif
```

dazu einfach eine Kopie von `user.ldif` verwenden als `modify.ldif`:
```ldif
dn: uid=john,ou=people,dc=example,dc=net
changeType: modify
replace: cn
cn: John C. Doe
```

# Access Control (ACL) managen

ACL-Konfiguration:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f acl.ldif
```

Inhalt von `acl.ldif`:
```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage by * break
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to dn.base="cn=Subschema" by * read
olcAccess: {3}to dn.subtree="dc=example,dc=net" by users read
```

Datenbanken auflistem:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config olcDatabase
```

ACLs kann als `root` über `ldapi:///` sehen:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b olcDatabase={1}mdb,cn=config olcAccess
```

Beispiel der ACL-Ausgabe:
```ldif
# {1}mdb, config
dn: olcDatabase={1}mdb,cn=config
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external
 ,cn=auth" manage by * break
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to dn.base="cn=Subschema" by * read
olcAccess: {3}to dn.subtree="dc=example,dc=net" by users read
olcAccess: {4}to attrs=userPassword by self write by anonymous auth by * none
olcAccess: {5}to attrs=shadowLastChange by self write by * read
olcAccess: {6}to * by * read
```

# TLS Konfiguration

generiere SSL-Zertifikat:
```bash
sudo apt install openssl
sudo openssl req -new -x509 -nodes -out /etc/ssl/certs/ldap.crt -keyout /etc/ldap/ldap.key -days 365
```
- entsprechende Abfragen beantworten

Berechtigungen prüfen und ggf. anpassen:
```bash
ls -l /etc/ldap/ldap.key
# gewünschter Zustand:
# -rw------- 1 openldap openldap ... /etc/ldap/ldap.key
sudo chown openldap:openldap /etc/ldap/ldap.key
sudo chmod 600 /etc/ldap/ldap.key
```

SSL-Zertifikat verwenden:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f tls.ldif
```

Inhalt von `tls.ldif`:
```ldif
dn: cn=config

changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/certs/ldap.crt

changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/certs/ldap.crt

dn: cn=config
changetype: modify
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/ldap.key
```

Zugriff über `ldaps` in `/etc/default/slapd` hinzufügen:
```bash
SLAPD_SERVICES="ldap:/// ldaps:/// ldapi:///"
```
- der Teil mit `ldaps:///` muss eingefügt werden

Dienst neu starten:
```bash
sudo systemctl restart slapd.service
```

Lauscher testen:
```bash
ss -tlnp | grep 636
```
- sollte eine Ausgabe erzeugen

SSL testen:
```bash
openssl s_client -connect ldap.example.net:636
```
- zeigt Details zur Verbindung an, wenn alles funktioniert

# Service Management

```bash
sudo systemctl start slapd
sudo systemctl status slapd
sudo systemctl restart slapd
```

# Konfiguration prüfen

```bash
sudo slaptest -v
```

# Weitere Themen und Befehle

- Loglevel konfigurieren
- `ldapwhoami` für auth testing
- Befehle aus `slapd`: `slapadd`
- Befehle aus `slapd`: `slapcat`
- Befehle aus `slapd`: `slapindex`
- **Client config**: Einstellungen in `/etc/ldap/ldap.conf`
